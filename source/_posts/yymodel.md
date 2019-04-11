title: YYModel 源码解析
date: 2019/4/9 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

YYModel 是一个高性能的 json model 解析库。

<!--more-->

## 使用

### Model 与 JSON 转换

```objc
// JSON:
{
    "uid":123456,
    "name":"Harry",
    "created":"1965-07-31T00:00:00+0000"
}

// Model:
@interface User : NSObject
@property UInt64 uid;
@property NSString *name;
@property NSDate *created;
@end
@implementation User
@end

	
// 将 JSON (NSData,NSString,NSDictionary) 转换为 Model:
User *user = [User yy_modelWithJSON:json];
	
// 将 Model 转换为 JSON 对象:
NSDictionary *json = [user yy_modelToJSONObject];
```

### Model 属性名和 JSON 中 key 不相同

1. model 属性对应于 JSON 中较深的 keypath
2. model 属性对应多个 JSON 中的 key

实现 JSON 转 Model 映射方法 `modelCustomPropertyMapper`

```objc
// JSON:
{
    "n":"Harry Pottery",
    "p": 256,
    "ext" : {
        "desc" : "A book written by J.K.Rowing."
    },
    "ID" : 100010
}

// Model:
@interface Book : NSObject
@property NSString *name;
@property NSInteger page;
@property NSString *desc;
@property NSString *bookID;
@end
@implementation Book
//返回一个 Dict，将 Model 属性名对映射到 JSON 的 Key。
+ (NSDictionary *)modelCustomPropertyMapper {
    return @{@"name" : @"n",
             @"page" : @"p",
             @"desc" : @"ext.desc",
             @"bookID" : @[@"id",@"ID",@"book_id"]};
}
@end
```

### Model 中包含其他 Model

直接使用自定义的类：

```objc
// JSON
{
    "author":{
        "name":"J.K.Rowling",
        "birthday":"1965-07-31T00:00:00+0000"
    },
    "name":"Harry Potter",
    "pages":256
}

// Model: 什么都不用做，转换会自动完成
@interface Author : NSObject
@property NSString *name;
@property NSDate *birthday;
@end
@implementation Author
@end
	
@interface Book : NSObject
@property NSString *name;
@property NSUInteger pages;
@property Author *author; //Book 包含 Author 属性
@end
@implementation Book
@end
```

### 容器类属性

数组和字典中存放的数据可以通过 `modelContainerPropertyGenericClass` 指定：

```objc
@class Shadow, Border, Attachment;

@interface Attributes
@property NSString *name;
@property NSArray *shadows; //Array<Shadow>
@property NSSet *borders; //Set<Border>
@property NSMutableDictionary *attachments; //Dict<NSString,Attachment>
@end

@implementation Attributes
// 返回容器类中的所需要存放的数据类型 (以 Class 或 Class Name 的形式)。
+ (NSDictionary *)modelContainerPropertyGenericClass {
    return @{@"shadows" : [Shadow class],
             @"borders" : Border.class,
             @"attachments" : @"Attachment" };
}
@end
```

### JSON 转 Model 完成后的操作

在 class 中实现相应方法，通过 `dic` 可以拿到 JSON 的字典，`self` 就是转换好的 Model：

```objc
- (id)modelCustomTransformFromDictionary:(NSDictionary *)dic  {
  // 操作 self
  return self
}
```

官方文档上没有提及这个方法，只是在我看源码的时候发现的。我觉得这个方法还是比较有用的，相当于在获取数据后在 Model 内执行类似于 watch 方法，可以把一些简单的数据处理放在 model 层。

### 额外功能：Coding/Copying/hash/equal/description 方法的实现

```objc
@interface YYShadow :NSObject <NSCoding, NSCopying>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) CGSize size;
@end

@implementation YYShadow
// 直接添加以下代码即可自动完成
- (void)encodeWithCoder:(NSCoder *)aCoder { [self yy_modelEncodeWithCoder:aCoder]; }
- (id)initWithCoder:(NSCoder *)aDecoder { self = [super init]; return [self yy_modelInitWithCoder:aDecoder]; }
- (id)copyWithZone:(NSZone *)zone { return [self yy_modelCopy]; }
- (NSUInteger)hash { return [self yy_modelHash]; }
- (BOOL)isEqual:(id)object { return [self yy_modelIsEqual:object]; }
- (NSString *)description { return [self yy_modelDescription]; }
@end
```

## 源码解析

### 文件结构

YYModel 的核心是以下两个文件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/yymodel_1.png?raw=true)

- `YYClassInfo` 主要是对 OC 中成员变量、属性、方法和类的一层封装。将一些需要通过 Runtime 方法获取到的属性保存在对象中，有助于减少相关方法调用此时，提高效率。
- `NSObject+YYModel` 提供了方法实现 model json 转换。

### YYClassInfo

YYClassInfo 中包含四个类，分别是对类及类中的成员标量、属性和方法的封装。

#### YYClassIvarInfo

`YYClassIvarInfo` 的属性及初始化方法如下：

```objc
@interface YYClassIvarInfo : NSObject
// 成员变量
@property (nonatomic, assign, readonly) Ivar ivar;
// 变量名称
@property (nonatomic, strong, readonly) NSString *name;
// 变量偏移量
@property (nonatomic, assign, readonly) ptrdiff_t offset;
// 变量类型编码
@property (nonatomic, strong, readonly) NSString *typeEncoding;
// 根据变量类型编码解析出来的相对应于自己的枚举
@property (nonatomic, assign, readonly) YYEncodingType type;
@end

@implementation YYClassIvarInfo
- (instancetype)initWithIvar:(Ivar)ivar {
    if (!ivar) return nil;
    self = [super init];
    _ivar = ivar;
    const char *name = ivar_getName(ivar);
    if (name) {
        _name = [NSString stringWithUTF8String:name];
    }
    _offset = ivar_getOffset(ivar);
    const char *typeEncoding = ivar_getTypeEncoding(ivar);
    if (typeEncoding) {
        _typeEncoding = [NSString stringWithUTF8String:typeEncoding];
        _type = YYEncodingGetType(typeEncoding);
    }
    return self;
}
@end
```

可以看到初始化方法就是把 `ivar` 的几个成员变量通过 Runtime 方法取出来，赋值给 `YYClassIvarInfo` 的相应属性中。

#### YYClassMethodInfo

```objc
@interface YYClassMethodInfo : NSObject
@property (nonatomic, assign, readonly) Method method;
@property (nonatomic, strong, readonly) NSString *name;
@property (nonatomic, assign, readonly) SEL sel;
@property (nonatomic, assign, readonly) IMP imp;
@property (nonatomic, strong, readonly) NSString *typeEncoding;
@property (nonatomic, strong, readonly) NSString *returnTypeEncoding;
@property (nullable, nonatomic, strong, readonly) NSArray<NSString *> *argumentTypeEncodings;
@end
```

这里就不再列举它的初始化方法了，总之就是把方法的相关信息提取到对象中。

####  YYClassPropertyInfo

属性中多了几个字段，包含属性的类，属性类遵守的协议以及属性的 get set 方法

```objc
@interface YYClassPropertyInfo : NSObject
@property (nonatomic, assign, readonly) objc_property_t property;
@property (nonatomic, strong, readonly) NSString *name;
@property (nonatomic, assign, readonly) YYEncodingType type;
@property (nonatomic, strong, readonly) NSString *typeEncoding;
@property (nonatomic, strong, readonly) NSString *ivarName;
@property (nullable, nonatomic, assign, readonly) Class cls;
@property (nullable, nonatomic, strong, readonly) NSArray<NSString *> *protocols;
@property (nonatomic, assign, readonly) SEL getter;
@property (nonatomic, assign, readonly) SEL setter;
```

#### YYClassInfo

```objc
@interface YYClassInfo : NSObject
@property (nonatomic, assign, readonly) Class cls;
@property (nullable, nonatomic, assign, readonly) Class superCls;
@property (nullable, nonatomic, assign, readonly) Class metaCls;
@property (nonatomic, readonly) BOOL isMeta;
@property (nonatomic, strong, readonly) NSString *name;
@property (nullable, nonatomic, strong, readonly) YYClassInfo *superClassInfo;
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassIvarInfo *> *ivarInfos;
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassMethodInfo *> *methodInfos;
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassPropertyInfo *> *propertyInfos;
@end
```

类封装中属性多了一些，包含了父类的 `YYClassInfo` 对象，以及属性、方法及成员变量的数组。

当然 `YYClassInfo` 中还有一些其他的细节，不是很重要就不在这里详述了。只要知道以上四个封装类即可。

### NSObject+YYModel

这个文件中实现了 Model和 JSON 的相互转换。不过这里先不直接开始转换方法的分析，而是先从与其相关的两个类 `_YYModelPropertyMeta` 以及 `_YYModelMeta` 开始。

#### _YYModelPropertyMeta

`_YYModelPropertyMeta` 是 `YYClassPropertyInfo` 的再上一层的封装。如果说 `YYClassPropertyInfo` 只是 `property_t` 的简单封装，那么 `_YYModelPropertyMeta` 就包含了 JSON 转 Model 所需的一些自定义配置。

```objc
@interface _YYModelPropertyMeta : NSObject {
    @package
    NSString *_name;
    YYEncodingType _type;
    YYEncodingNSType _nsType;
    YYClassPropertyInfo *_info;
    BOOL _isCNumber;
    Class _cls;
    SEL _getter;
    SEL _setter;
    BOOL _isKVCCompatible;  	
    BOOL _isStructAvailableForKeyedArchiver;
  
  	// 属性是否有自定义的类型。一般情况下不会出现要自定义类型的情况，因为这样的方式自定义类型不如直接创建新的类型直接。文档上也没有相关说明。因此后面这部分相关的代码直接略过。
    BOOL _hasCustomClassFromDictionary;
  	// 一般用于字典数组等容器类中。表示容器的泛型类型
    Class _genericCls;
		// 当前属性对应于 JSON 中的 key 或者 keyPath
    NSString *_mappedToKey;
    NSArray *_mappedToKeyPath;
    NSArray *_mappedToKeyArray;
  	// 如果 JSON 中一个 key 对应于 model 多个键的情况下，通过这个指针保存
    _YYModelPropertyMeta *_next;
}
@end
```



#### _YYModelMeta

##### _YYModelMeta 的属性

`_YYModelMeta` 是 `YYClassInfo` 的上层封装。属性如下：

```objc
@interface _YYModelMeta : NSObject {
    @package
    YYClassInfo *_classInfo;
    // 键是属性名，值是属性的一个封装 YYModelPropertyMeta
    NSDictionary *_mapper;
  	// 所有的 YYModelPropertyMeta 的集合
    NSArray *_allPropertyMetas;
  	// 如果 class 实现了 modelCustomPropertyMapper 方法，其中有 JSON 的 keypaths 对应于一个 model 的键的情况，那么会被加入到这个数组中
    NSArray *_keyPathPropertyMetas;
  	// 如果 class 实现了 modelCustomPropertyMapper 方法，其中可能 JSON 中多个字段对应于一个 model 的键的情况，那么就会加入到这个数组中
    NSArray *_multiKeysPropertyMetas;
    NSUInteger _keyMappedCount;
    YYEncodingNSType _nsType;
    
  	// 如果实现了相应的方法，则在 JSON 转换 Model 后可以做一些自定义的操作
    BOOL _hasCustomWillTransformFromDictionary;
    BOOL _hasCustomTransformFromDictionary;
    BOOL _hasCustomTransformToDictionary;
    BOOL _hasCustomClassFromDictionary;
}
@end
```

##### 获取 _YYModelMeta 的方法

通过下面的方法获取 `_YYModelMeta`。`_YYModelMeta` 是类级的对象，一个类型可以共享一个实例。因此，`_YYModelMeta` 实例被保存在一个单例的 cache 字典中，只有该类型未被初始化的时候才会通过 `initWithClass` 方法创建：

```objc
+ (instancetype)metaWithClass:(Class)cls {
    if (!cls) return nil;
    static CFMutableDictionaryRef cache;
    static dispatch_once_t onceToken;
    static dispatch_semaphore_t lock;
    dispatch_once(&onceToken, ^{
      	// _YYModelMeta 的缓存字典
        cache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
  	// 通过外部传进来的 class 来获取相应的 _YYModelMeta 实例。
    _YYModelMeta *meta = CFDictionaryGetValue(cache, (__bridge const void *)(cls));
    dispatch_semaphore_signal(lock);
    if (!meta || meta->_classInfo.needUpdate) {
     		// 
        meta = [[_YYModelMeta alloc] initWithClass:cls];
        if (meta) {
            dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
            CFDictionarySetValue(cache, (__bridge const void *)(cls), (__bridge const void *)(meta));
            dispatch_semaphore_signal(lock);
        }
    }
    return meta;
}
```

##### 创建 _YYModelMeta

`_YYModelMeta` 的初始化方法比较长，这里截取一段段代码分别分析。

**容器类型的设置**

```objc
// 将 modelContainerPropertyGenericClass 方法返回的泛型
NSDictionary *genericMapper = nil;
if ([cls respondsToSelector:@selector(modelContainerPropertyGenericClass)]) {
    genericMapper = [(id<YYModel>)cls modelContainerPropertyGenericClass];
    if (genericMapper) {
        NSMutableDictionary *tmp = [NSMutableDictionary new];
        [genericMapper enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            if (![key isKindOfClass:[NSString class]]) return;
            Class meta = object_getClass(obj);
            if (!meta) return;
            if (class_isMetaClass(meta)) {
                tmp[key] = obj;
            } else if ([obj isKindOfClass:[NSString class]]) {
              	// 如果是字符串类型，那么转为 class 类型
                Class cls = NSClassFromString(obj);
                if (cls) {
                    tmp[key] = cls;
                }
            }
        }];
        genericMapper = tmp;
    }
}
```

这个方法用来处理容器类属性。当 Model 实现了 `modelContainerPropertyGenericClass` 方法的时候(如果不知道，可以看上面的使用介绍)，就将结果暂时保存在 `genericMapper` 中，待到后面实例化 `_YYModelPropertyMeta` 对象的时候设置到  `class genericCls` 中。

**Model 对应 JSON keyPaths 的设置**

相关实现是通过在 Model 中实现 `modelCustomPropertyMapper` 实现的。如果 `modelCustomPropertyMapper` 返回的字典中值是一个带有 `.` 的字符串，那么就说明 Model 对应 JSON 中的 keyPaths：

```objc
// 在 model 中实现了 modelCustomPropertyMapper 的情况
if ([cls respondsToSelector:@selector(modelCustomPropertyMapper)]) {
    NSDictionary *customMapper = [(id <YYModel>)cls modelCustomPropertyMapper];
    [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
      	// 从 allPropertyMetas 中找到方法返回值相关的属性
        _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
        if (!propertyMeta) return;
      	// 由于要自定义设置，所以先删除这个属性相关的 _YYModelPropertyMeta 实例
        [allPropertyMetas removeObjectForKey:propertyName];
        
      	// 如果是字符串就表示是要将 JSON 中的 keypath 映射到 Modal 中
        if ([mappedToKey isKindOfClass:[NSString class]]) {
            if (mappedToKey.length == 0) return;
            
            propertyMeta->_mappedToKey = mappedToKey;
          	// 以 '.' 作为分割取出数组
            NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
            for (NSString *onePath in keyPath) {
                if (onePath.length == 0) {
                    NSMutableArray *tmp = keyPath.mutableCopy;
                    [tmp removeObject:@""];
                    keyPath = tmp;
                    break;
                }
            }
            if (keyPath.count > 1) {
              	// 将 keyPath 赋值给 _YYModelPropertyMeta 的 _mappedToKeyPath 属性
                propertyMeta->_mappedToKeyPath = keyPath;
                [keyPathPropertyMetas addObject:propertyMeta];
            }
          	// 如果 mapper 中已经存在该 mappedToKey 这个键值对，那么说明 mappedToKey 这个 keypath 将对应多个 model 中的键。通过 _next 标识(写的比较拗口，如果不明白也没关系，这个不是重点)
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
            mapper[mappedToKey] = propertyMeta;
        }
    }
}

// 保存到 _YYModelMeta 的 _keyPathPropertyMetas 中
if (keyPathPropertyMetas) _keyPathPropertyMetas = keyPathPropertyMetas;
```

**Model 对应 JSON 中多个可能的字段的设置**

还是 `modelCustomPropertyMapper` 中实现的。如果返回的字典中的值是一个数组，那么说明 Model 中的属性可能对应 JSON 中多个可能的字段：

```objc
if ([cls respondsToSelector:@selector(modelCustomPropertyMapper)]) {
    NSDictionary *customMapper = [(id <YYModel>)cls modelCustomPropertyMapper];
    [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
        _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
        if (!propertyMeta) return;
        [allPropertyMetas removeObjectForKey:propertyName];
        
      	// 判断 modelCustomPropertyMapper 返回的字典中的每一个值是不是一个数组
        if ([mappedToKey isKindOfClass:[NSArray class]]) {
            NSMutableArray *mappedToKeyArray = [NSMutableArray new];
          	// 如果是数组，遍历数组中的每一项
            for (NSString *oneKey in ((NSArray *)mappedToKey)) {
                if (![oneKey isKindOfClass:[NSString class]]) continue;
                if (oneKey.length == 0) continue;
                // 如果其中包含 '.'，那么和之前一样，表明 Model 中的这个键对应于 JSON 中的 keyPaths
                NSArray *keyPath = [oneKey componentsSeparatedByString:@"."];
                if (keyPath.count > 1) {
                    [mappedToKeyArray addObject:keyPath];
                } else {
                    [mappedToKeyArray addObject:oneKey];
                }
                
                if (!propertyMeta->_mappedToKey) {
                    propertyMeta->_mappedToKey = oneKey;
                    propertyMeta->_mappedToKeyPath = keyPath.count > 1 ? keyPath : nil;
                }
            }
            if (!propertyMeta->_mappedToKey) return;
            
            propertyMeta->_mappedToKeyArray = mappedToKeyArray;
          	// NSArray 中解析好的每一项添加到 mutiKeysPropertyMetas 中
            [multiKeysPropertyMetas addObject:propertyMeta];
            
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
            mapper[mappedToKey] = propertyMeta;
        }
    }];
}

// 保存到 _YYModelMeta 中的 _multiKeysPropertyMetas 中
if (multiKeysPropertyMetas) _multiKeysPropertyMetas = multiKeysPropertyMetas;
```

### JSON → Model 的实现

#### 外部调用方法

上面把用到的数据结构都介绍了一遍，现在开始正真的 JSON → Model 的转换。这里省略了 JSON → NSDictionary 的转换(通过 `NSJSONSerialization` 实现)。直接看 NSDictionary → Model 的实现：

```objc
+ (instancetype)yy_modelWithDictionary:(NSDictionary *)dictionary {
    if (!dictionary || dictionary == (id)kCFNull) return nil;
    if (![dictionary isKindOfClass:[NSDictionary class]]) return nil;
    
    Class cls = [self class];

  	// 创建一个实例 one，然后通过 yy_modelSetWithDictionary 方法真正的将字典中的键值对填充到对象中
    NSObject *one = [cls new];
    if ([one yy_modelSetWithDictionary:dictionary]) return one;
    return nil;
}


- (BOOL)yy_modelSetWithDictionary:(NSDictionary *)dic {
 		...
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:object_getClass(self)];
    
  	// 上下文对象，将 Model 和 Dictionary 都保存在这个对象中，这样处理方法就可以直接通过这个 context 拿到 Model 和 Dictionary
    ModelSetContext context = {0};
    context.modelMeta = (__bridge void *)(modelMeta);
    context.model = (__bridge void *)(self);
    context.dictionary = (__bridge void *)(dic);
    
    // 一般情况下，Model 中的属性和字典中的字段是一一对应的，但是如果 Model 中的属性多余字典中的字段，那么说明 Model 中的属性可能映射了字典中某个值中的某个字段，即对应的是一个 keyPaths
    if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {
      	// Dictionary 转 Model。CFDictionaryApplyFunction 就是对 Dictionary 中的每个键都执行传入的方法
        CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
      	// 如果 Model 可能存在 keypaths 的映射，那么执行，将 keypaths 对应的值
        if (modelMeta->_keyPathPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_keyPathPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_keyPathPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
        if (modelMeta->_multiKeysPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_multiKeysPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_multiKeysPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
    } else {
        CFArrayApplyFunction((CFArrayRef)modelMeta->_allPropertyMetas,
                             CFRangeMake(0, modelMeta->_keyMappedCount),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }
    
    /// JSON 转 model 转换完成后，自定义的转换部分
    if (modelMeta->_hasCustomTransformFromDictionary) {
        return [((id<YYModel>)self) modelCustomTransformFromDictionary:dic];
    }
    return YES;
}
```

可以看到，上面的设置过程主要涉及了两个方法。其中第一个是 JSON 转 Modal 的核心：

- ModelSetWithDictionaryFunction
- ModelSetWithPropertyMetaArrayFunction

#### ModelSetWithDictionaryFunction

这个方法传入的三个参数分别为，JSON 转成的 Dictionary 的每个键值对和传入的上下文。

```objc
static void ModelSetWithDictionaryFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContext *context = _context;
  	// 通过上下文拿到 Class 的封装 _YYModelMeta
    __unsafe_unretained _YYModelMeta *meta = (__bridge _YYModelMeta *)(context->modelMeta);
  	// 从 _YYModelMeta 中取出 key 对应的 _YYModelPropertyMeta 
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = [meta->_mapper objectForKey:(__bridge id)(_key)];
  	// 从上下文中拿到 model 实例
    __unsafe_unretained id model = (__bridge id)(context->model);
    /// 看看 key 对应的属性有没有 setter 方法，如果没有，那么就通过 next 找
    while (propertyMeta) {
        if (propertyMeta->_setter) {
          	// 核心方法，将 value 设置到 model 中
            ModelSetValueForProperty(model, (__bridge __unsafe_unretained id)_value, propertyMeta);
        }
        propertyMeta = propertyMeta->_next;
    };
}
```

最核心的方法就是 `ModelSetValueForProperty`，真正将值赋值给 model 就是在这个方法中。它的处理代码非常长，因为针对不同类型的属性要做不同的处理。一般情况下，可以将处理代码分为三类：

1. c 中的数字类型。如 int 等
2. oc 中的对象类型。如 NSString，NSNumber 等
3. 除以上两种的其他类型。如 block 类型，SEL 类型

#### C 类型设置

C 类型值设置方法如下：

```objc
if (meta->_isCNumber) {
  	// 把 Value 转为 NSNumber 类型
    NSNumber *num = YYNSNumberCreateFromID(value);
    ModelSetNumberToProperty(model, num, meta);
    if (num != nil) [num class]; // hold the number
}
```

先把 Value 转为 NSNumber 类型，然后调用 `ModelSetNumberToProperty` 方法。摘取一部分代码如下：

```objc
static force_inline void ModelSetNumberToProperty(__unsafe_unretained id model,
                                                  __unsafe_unretained NSNumber *num,
                                                  __unsafe_unretained _YYModelPropertyMeta *meta) {
    switch (meta->_type & YYEncodingTypeMask) {
        case YYEncodingTypeBool: {
            ((void (*)(id, SEL, bool))(void *) objc_msgSend)((id)model, meta->_setter, num.boolValue);
        } break;
        ...
        default: break;
    }
}
```

可以看到，最终是直接调用了 `objc_msgSend` 执行 model 相应属性的 setter 方法。

#### OC 类型设置

```objc
if (meta->_nsType) {
    if (value == (id)kCFNull) {
        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, (id)nil);
    } else {
        switch (meta->_nsType) {
            case YYEncodingTypeNSString:
            case YYEncodingTypeNSMutableString: {
                if ([value isKindOfClass:[NSString class]]) {
                    if (meta->_nsType == YYEncodingTypeNSString) {
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, value);
                    } else {
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, ((NSString *)value).mutableCopy);
                    }
                } else if ([value isKindOfClass:[NSNumber class]]) {
                    ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                    meta->_setter,
                                                                    (meta->_nsType == YYEncodingTypeNSString) ?
                                                                    ((NSNumber *)value).stringValue :
                                                                    ((NSNumber *)value).stringValue.mutableCopy);
                } else if ([value isKindOfClass:[NSData class]]) {
                    NSMutableString *string = [[NSMutableString alloc] initWithData:value encoding:NSUTF8StringEncoding];
                    ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, string);
                } else if ([value isKindOfClass:[NSURL class]]) {
                    ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                    meta->_setter,
                                                                    (meta->_nsType == YYEncodingTypeNSString) ?
                                                                    ((NSURL *)value).absoluteString :
                                                                    ((NSURL *)value).absoluteString.mutableCopy);
                } else if ([value isKindOfClass:[NSAttributedString class]]) {
                    ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                    meta->_setter,
                                                                    (meta->_nsType == YYEncodingTypeNSString) ?
                                                                    ((NSAttributedString *)value).string :
                                                                    ((NSAttributedString *)value).string.mutableCopy);
                }
            } break;
            ...
        }
    }
}
```

这里截取的代码有点长。不过已经是精简了很多的状态了。总体而言还是判断属性的类型，然后调用相应的 `objc_msgSend` 方法。

### Model → JSONObject 的实现

Model 到 JSONObject 的转换也就是 Model 到数组或者字典这种模型的转换，核心方法在递归方法 `ModelToJSONObjectRecursive` 。

```objc
- (id)yy_modelToJSONObject {
    // 递归转换模型到 JSON
    id jsonObject = ModelToJSONObjectRecursive(self);
    if ([jsonObject isKindOfClass:[NSArray class]]) return jsonObject;
    if ([jsonObject isKindOfClass:[NSDictionary class]]) return jsonObject;
    
    return nil;
}
```

具体的转换代码非常长。做了注释后还是贴上来了：

```objc
// 递归转换模型到 JSON，如果转换异常则返回 nil
static id ModelToJSONObjectRecursive(NSObject *model) {
    // 判空或者可以直接返回的对象，则直接返回
    if (!model || model == (id)kCFNull) return model;
    if ([model isKindOfClass:[NSString class]]) return model;
    if ([model isKindOfClass:[NSNumber class]]) return model;
    // 如果 model 从属于 NSDictionary
    if ([model isKindOfClass:[NSDictionary class]]) {
        // 如果可以直接转换为 JSON 数据，则返回
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        NSMutableDictionary *newDic = [NSMutableDictionary new];
        // 遍历 model 的 key 和 value
        [((NSDictionary *)model) enumerateKeysAndObjectsUsingBlock:^(NSString *key, id obj, BOOL *stop) {
            NSString *stringKey = [key isKindOfClass:[NSString class]] ? key : key.description;
            if (!stringKey) return;
            // 递归解析 value 
            id jsonObj = ModelToJSONObjectRecursive(obj);
            if (!jsonObj) jsonObj = (id)kCFNull;
            newDic[stringKey] = jsonObj;
        }];
        return newDic;
    }
    // 如果 model 从属于 NSSet
    if ([model isKindOfClass:[NSSet class]]) {
        // 如果能够直接转换 JSON 对象，则直接返回
        // 否则遍历，按需要递归解析
        ...
    }
    if ([model isKindOfClass:[NSArray class]]) {
        // 如果能够直接转换 JSON 对象，则直接返回
        // 否则遍历，按需要递归解析
        ...
    }
    // 对 NSURL, NSAttributedString, NSDate, NSData 做相应处理
    if ([model isKindOfClass:[NSURL class]]) return ((NSURL *)model).absoluteString;
    if ([model isKindOfClass:[NSAttributedString class]]) return ((NSAttributedString *)model).string;
    if ([model isKindOfClass:[NSDate class]]) return [YYISODateFormatter() stringFromDate:(id)model];
    if ([model isKindOfClass:[NSData class]]) return nil;
    
    // 用 [model class] 初始化一个模型元
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:[model class]];
    // 如果映射表为空，则不做解析直接返回 nil
    if (!modelMeta || modelMeta->_keyMappedCount == 0) return nil;
    // 性能优化细节，使用 __unsafe_unretained 来避免在下面遍历 block 中直接使用 result 指针造成的不必要 retain 与 release 开销
    NSMutableDictionary *result = [[NSMutableDictionary alloc] initWithCapacity:64];
    __unsafe_unretained NSMutableDictionary *dic = result;
    // 遍历模型元属性映射字典
    [modelMeta->_mapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyMappedKey, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
        // 如果遍历当前属性元没有 getter 方法，跳过
        if (!propertyMeta->_getter) return;
        
        id value = nil;
        // 如果属性元属于 CNumber，即其 type 是 int、float、double 之类的
        if (propertyMeta->_isCNumber) {
            // 从属性中利用 getter 方法得到对应的值
            value = ModelCreateNumberFromProperty(model, propertyMeta);
        } else if (propertyMeta->_nsType) { // 属性元属于 nsType，即 NSString 之类
            // 利用 getter 方法拿到 value
            id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
            // 对拿到的 value 递归解析
            value = ModelToJSONObjectRecursive(v);
        } else {
            // 根据属性元的 type 做相应处理
            switch (propertyMeta->_type & YYEncodingTypeMask) {
                // id，需要递归解析，如果解析失败则返回 nil
                case YYEncodingTypeObject: {
                    id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = ModelToJSONObjectRecursive(v);
                    if (value == (id)kCFNull) value = nil;
                } break;
                // Class，转 NSString，返回 Class 名称
                case YYEncodingTypeClass: {
                    Class v = ((Class (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromClass(v) : nil;
                } break;
                // SEL，转 NSString，返回给定 SEL 的字符串表现形式
                case YYEncodingTypeSEL: {
                    SEL v = ((SEL (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromSelector(v) : nil;
                } break;
                default: break;
            }
        }
        // 如果 value 还是没能解析，则跳过
        if (!value) return;
        
        // 当前属性元是 KeyPath 映射，即 a.b.c 之类
        if (propertyMeta->_mappedToKeyPath) {
            NSMutableDictionary *superDic = dic;
            NSMutableDictionary *subDic = nil;
            // _mappedToKeyPath 是 a.b.c 根据 '.' 拆分成的字符串数组，遍历 _mappedToKeyPath
            for (NSUInteger i = 0, max = propertyMeta->_mappedToKeyPath.count; i < max; i++) {
                NSString *key = propertyMeta->_mappedToKeyPath[i];
                // 遍历到结尾
                if (i + 1 == max) {
                    // 如果结尾的 key 为 nil，则使用 value 赋值
                    if (!superDic[key]) superDic[key] = value;
                    break;
                }
                
                // 用 subDic 拿到当前 key 对应的值
                subDic = superDic[key];
                // 如果 subDic 存在
                if (subDic) {
                    // 如果 subDic 从属于 NSDictionary
                    if ([subDic isKindOfClass:[NSDictionary class]]) {
                        // 将 subDic 的 mutable 版本赋值给 superDic[key]
                        subDic = subDic.mutableCopy;
                        superDic[key] = subDic;
                    } else {
                        break;
                    }
                } else {
                    // 将 NSMutableDictionary 赋值给 superDic[key]
                    // 注意这里使用 subDic 间接赋值是有原因的，原因就在下面
                    subDic = [NSMutableDictionary new];
                    superDic[key] = subDic;
                }
                // superDic 指向 subDic，这样在遍历 _mappedToKeyPath 时即可逐层解析
                // 这就是上面先把 subDic 转为 NSMutableDictionary 的原因
                superDic = subDic;
                subDic = nil;
            }
        } else {
            // 如果不是 KeyPath 则检测 dic[propertyMeta->_mappedToKey]，如果为 nil 则赋值 value
            if (!dic[propertyMeta->_mappedToKey]) {
                dic[propertyMeta->_mappedToKey] = value;
            }
        }
    }];
    
    // 忽略，对应 modelCustomTransformToDictionary 接口
    if (modelMeta->_hasCustomTransformToDictionary) {
        // 用于在默认的 Model 转 JSON 过程不适合当前 Model 类型时提供自定义额外过程
        // 也可以用这个方法来验证转换结果
        BOOL suc = [((id<YYModel>)model) modelCustomTransformToDictionary:dic];
        if (!suc) return nil;
    }
    
    return result;
}
```

总的来说，还是判断传入的 Model 的类型，如果是 NSString 这种基本类型，那么就直接返回了。如果是 class 类型，那么就生成 class 对应的 `_YYModelMeta` 对象，然后遍历其中的所有属性，将其赋值给 `result` 这个字典中。

## 一些点

### 设值

一般情况下，我们都会认为字典模型转换都是使用 KVC 的方式。然鹅 YYModel 并不是使用 KVC，而是通过 `objc_msgSend` 调用 Model 中 property 的 setter 方法。这样速度上更快。

### 内存优化

`__unsafe_unretained` 在作用上和 `__weak` 类似，但是 `__unsafe_unretained` 会更易造成野指针。

 访问具有 `__weak` 属性的变量时，实际上会调用 `objc_loadWeak()` 和 `objc_storeWeak()` 来完成，这也会带来很大的开销，所以如果你能确定在该变量的使用过程中，不会被回收，就可以使用 `__unsafe_unretained` 替代 `__weak` 。

### 使用 c 函数

YYModel 中使用了大量的 c 函数。使用纯 C 函数可以避免 ObjC 的消息发送带来的开销。

### 使用内联函数

内联函数用 `inline` 标记。编译时，类似宏替换，使用函数体替换调用处的函数名。

优势是内联减少了函数调用的开销，在调用时不发生控制转移。劣势是如果调用次数很多时，一般会增加了代码量。

内联函数和宏定义相比差别在于：

- 编译阶段：宏定义使用预处理器preprocessor实现，只是预处理符号表的简单替换。内联函数则在编译阶段进行替换，会有类型检查和参数有效性的检测。
- 安全性：内联函数具有类型检查与参数有效性的验证，而宏没有；

## 总结

总的来说，YYModel 为了效率使用了很多 c 函数，其实在日常的编程中是不太能用到的(或者说，你要这么写是会被骂的)，但正是这些更底层不常用的函数使 YYModel 成为了一个高效的 JSON Model 转换库。



## 参考

[揭秘 YYModel 的魔法](https://juejin.im/post/5a097435f265da431769a49c)

[解码YYModel（一）基础](http://wenghengcong.com/posts/ec42f57/)

[YYModel](https://github.com/ibireme/YYModel)