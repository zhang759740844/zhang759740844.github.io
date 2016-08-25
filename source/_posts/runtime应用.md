title: runtime应用笔记
date: 2016/8/23 14:07:12  
categories: IOS
tags: [runtime]

---

了解了runtime的基本原理，那么runtime究竟有什么用处呢？参考[Runtime全方位装逼指南](http://www.jianshu.com/p/efeb33712445)，总结了以下几点应用场景。

<!--more-->

## 给category添加属性
### 原理
**对象关联**允许开发者对已经存在的类在 Category 中添加自定义的属性：
```objc
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);
```
- object 是源对象.
- value 是被关联的对象.
- key 是关联的键，objc_getAssociatedObject 方法通过不同的 key 即可取出对应的被关联对象.
- policy 是一个枚举值，表示关联对象的行为，从命名就能看出各个枚举值的含义：
```objc
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

要取出被关联的对象使用 objc_getAssociatedObject 方法即可，要删除一个被关联的对象，使用 objc_setAssociatedObject 方法将对应的 key 设置成 nil 即可：
```objc
objc_setAssociatedObject(self, associatedKey, nil, OBJC_ASSOCIATION_COPY_NONATOMIC);
```

objc_removeAssociatedObjects 方法将会移除源对象中所有的关联对象.

### 示例：
新建 UIButton 的 Category，在其中设置clickBlock属性(UIButton+ClickBlock.h):
```objc
typedef void(^clickBlock)(void);

@interface UIButton (ClickBlock)
@property (nonatomic,copy) clickBlock click;
@end
```

在.m中设置click的set，get方法(UIButton+ClickBlock.m):
```objc
#import "UIButton+ClickBlock.h"
#import <objc/runtime.h>

static const void *associatedKey = "associatedKey";

@implementation UIButton (ClickBlock)

//Category中的属性，只会生成setter和getter方法，不会生成成员变量

-(void)setClick:(clickBlock)click{
    objc_setAssociatedObject(self, associatedKey, click, OBJC_ASSOCIATION_COPY_NONATOMIC);
    [self removeTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    if (click) {
        [self addTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    }
}

-(clickBlock)click{
    return objc_getAssociatedObject(self, associatedKey);
}

-(void)buttonClick{
    if (self.click) {
        self.click();
    }
}

@end
```

其中，在set方法中使用`addTarget:action:forControlEvents:`给button设置了点击事件。

`self.click()`表示使用`self.click`获得block，再通过`block()`执行块。

为什么不直接给click赋值，而是通过runtime的`objc_setAssociatedObject`方法呢？
@property属性会在编译时自动生成一个实例变量_click以及其set，get方法。但是由于runtime只能添加方法，不能添加实例变量。因此，_click并没有添加进UIButton的ivar中，因而不能使用。只能通过runtime的方法，添加对应键值对。

@property在本例中只是为了在.h里声明一个getset方法。可替换成：
```objc
typedef void(^clickBlock)(void);

@interface UIButton (ClickBlock)
//@property (nonatomic,copy) clickBlock click;
- (clickBlock)click;
- (void)setClick:(clickBlock)click;
@end
```

## 字典与模型转换
### 原理
字典转模型的时候：
1. 根据字典的 key 生成 setter 方法.
2. 使用 objc_msgSend 调用 setter 方法为 Model 的属性赋值（或者 KVC）.

模型转字典的时候：
1. 调用 class_copyPropertyList 方法获取当前 Model 的所有属性.
2. 调用 property_getName 获取属性名称.
3. 根据属性名称生成 getter 方法.
4. 使用 objc_msgSend 调用 getter 方法获取属性值（或者 KVC）.

### 示例
```objc
@interface NSObject (KeyValues)

+(id)objectWithKeyValues:(NSDictionary *)aDictionary;

-(NSDictionary *)keyValuesWithObject;

@end


#import "NSObject+KeyValues.h"
#import <objc/runtime.h>
#import <objc/message.h>

@implementation NSObject (KeyValues)

//字典转模型
+(id)objectWithKeyValues:(NSDictionary *)aDictionary{
    id objc = [[self alloc] init];
    for (NSString *key in aDictionary.allKeys) {
        id value = aDictionary[key];
        
        /*判断当前属性是不是Model*/
        objc_property_t property = class_getProperty(self, key.UTF8String);
        unsigned int outCount = 0;
        objc_property_attribute_t *attributeList = property_copyAttributeList(property, &outCount);
        objc_property_attribute_t attribute = attributeList[0];
        NSString *typeString = [NSString stringWithUTF8String:attribute.value];
        if ([typeString isEqualToString:@"@\"TestModel\""]) {
            value = [self objectWithKeyValues:value];
        }
        /**********************/
        
        //生成setter方法，并用objc_msgSend调用
        NSString *methodName = [NSString stringWithFormat:@"set%@%@:",[key substringToIndex:1].uppercaseString,[key substringFromIndex:1]];
        SEL setter = sel_registerName(methodName.UTF8String);
        if ([objc respondsToSelector:setter]) {
            ((void (*) (id,SEL,id)) objc_msgSend) (objc,setter,value);
        }
        free(attributeList);
    }
    return objc;
}

//模型转字典
-(NSDictionary *)keyValuesWithObject{
    unsigned int outCount = 0;
    objc_property_t *propertyList = class_copyPropertyList([self class], &outCount);
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    for (int i = 0; i < outCount; i ++) {
        objc_property_t property = propertyList[i];
        
        //生成getter方法，并用objc_msgSend调用
        const char *propertyName = property_getName(property);
        SEL getter = sel_registerName(propertyName);
        if ([self respondsToSelector:getter]) {
            id value = ((id (*) (id,SEL)) objc_msgSend) (self,getter);
            
            /*判断当前属性是不是Model*/
            if ([value isKindOfClass:[self class]] && value) {
                value = [value keyValuesWithObject];
            }
            /**********************/
            
            if (value) {
                NSString *key = [NSString stringWithUTF8String:propertyName];
                [dict setObject:value forKey:key];
            }
        }
        
    }
    free(propertyList);
    return dict;
}
@end
```

使用：
```objc
-(void)keyValuesTest{
    
    TestModel *model = [TestModel objectWithKeyValues:dictionary];
    NSLog(@"name is %@",model.name);
    NSLog(@"son name is %@",model.son.name);
    
    NSDictionary *dict = [model keyValuesWithObject];
    NSLog(@"dict is %@",dict);
}
```

注意：
1. 在NSObject中添加类方法，其中的`self`指的是`TestModel`这个类。
2. `objc_property_t`具有两个属性，name和attribute。调用`property_getAttribute`将返回attribute的字符串。调用`property_copyAttributeList`则将字符串切分，返回一个`objc_property_attribute_t`类型的指针，`outCount`返回了属性的数量。

---
outCount使用了**指向指针的指针**的方式，使没有返回outCount的情况下，修改了outCount的值。

例：
```objc
- (NSArray *)scanBeginStr:(NSString *)beginstr endStr:(NSString *)endstr inText:(NSMutableString * *)textPointer{
    NSRange range1,range2;
    NSUInteger location =0,length=0;
    range1.location = 0;
    NSMutableString *text = *textPointer;
    NSMutableArray *rangeArray = [NSMutableArray array];
    while (range1.location != NSNotFound) {
        range1 = [text rangeOfString:beginstr];
        range2 = [text rangeOfString:endstr];
        if (range1.location != NSNotFound) {
            location = range1.location;
            length = range2.location - range1.location - 1;
            if (length > 5000)break;
            [text replaceOccurrencesOfString:beginstr withString:@"" options:NSCaseInsensitiveSearch range:NSMakeRange(0, range1.location + range1.length)];
            [text replaceOccurrencesOfString:endstr withString:@"" options:NSCaseInsensitiveSearch range:NSMakeRange(0, range2.location + range2.length - 1)];
        }
        [rangeArray addObject:@{@"location":@(location),@"length":@(length)}];
    }
    return rangeArray;
}
```
使用：通过&取指针的地址
```
 NSArray *rangeArray = [self scanBegin3Str:@"<" endStr:@">" inText:&mutableText];
```

---

## 自动归档
### 原理
归档是一种很常用的文件储存方法，几乎任何类型的对象都能够被归档储存。自动归档就是动态获取model的各个属性，进行保存：
```objc
- (void)encodeWithCoder:(NSCoder *)aCoder{
    [aCoder encodeObject:self.name forKey:@"name"];
    [aCoder encodeObject:self.ID forKey:@"ID"];
}

- (id)initWithCoder:(NSCoder *)aDecoder{
    if (self = [super init]) {
        self.ID = [aDecoder decodeObjectForKey:@"ID"];
        self.name = [aDecoder decodeObjectForKey:@"name"];
    }
    return self;
}
```
如果当前 Model 有100个属性的话，就需要写100行这种代码.通过 Runtime 我们就可以轻松解决这个问题：
1. 使用 class_copyIvarList 方法获取当前 Model 的所有成员变量.
2. 使用 ivar_getName 方法获取成员变量的名称.
3. 通过 KVC 来读取 Model 的属性值（encodeWithCoder:），以及给 Model 的属性赋值（initWithCoder:）.

### 示例：
```objc
#import "TestModel.h"
#import <objc/runtime.h>
#import <objc/message.h>

@implementation TestModel
- (void)encodeWithCoder:(NSCoder *)aCoder{
    unsigned int outCount = 0;
    Ivar *vars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar var = vars[i];
        const char *name = ivar_getName(var);
        NSString *key = [NSString stringWithUTF8String:name];
        
        // 注意kvc的特性是，如果能找到key这个属性的setter方法，则调用setter方法
        // 如果找不到setter方法，则查找成员变量key或者成员变量_key，并且为其赋值
        // 所以这里不需要再另外处理成员变量名称的“_”前缀
        id value = [self valueForKey:key];
        [aCoder encodeObject:value forKey:key];
    }
    free(vars);
}

- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder{
    if (self = [super init]) {
        unsigned int outCount = 0;
        Ivar *vars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar var = vars[i];
            const char *name = ivar_getName(var);
            NSString *key = [NSString stringWithUTF8String:name];
            id value = [aDecoder decodeObjectForKey:key];
            [self setValue:value forKey:key];
        }
        free(vars);
    }
    return self;
}
@end
```

使用：
```objc
-(void)keyedArchiverTest{
    
    TestModel *model = [TestModel objectWithKeyValues:dictionary];
    
    NSString *path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
    path = [path stringByAppendingPathComponent:@"test"];
    [NSKeyedArchiver archiveRootObject:model toFile:path];
    
    TestModel *m = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
    NSLog(@"m.name is %@",m.name);
    NSLog(@"m.son name is %@",m.son.name);
}
```

## 动态方法解析
### 原理
如果某个对象调用了不存在的方法时，一般情况下程序会crash.但是在程序crash之前，Runtime 会给我们动态方法解析的机会，消息发送的步骤大致如下：
1. 检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain，release 这些函数了.
2. 检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉.
3. 如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行.如果 cache 找不到就找一下方法分发表.
4. 如果分发表找不到就到超类的分发表去找，一直找，直到找到NSObject类为止.
5. 如果还找不到就要开始进入消息转发了，消息转发的大致过程如图：

![消息转发流程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_dynamicMethod.jpg?raw=true)

1. 进入 resolveInstanceMethod: 方法，指定是否动态添加方法。若返回NO，则进入下一步，若返回YES，则通过 class_addMethod 函数动态地添加方法，消息得到处理，此流程完毕.
2. resolveInstanceMethod: 方法返回 NO 时，就会进入 forwardingTargetForSelector: 方法，这是 Runtime 给我们的第二次机会，用于指定哪个对象响应这个 selector。返回nil，进入下一步，返回某个对象，则会调用该对象的方法.
3. 若 forwardingTargetForSelector: 返回的是nil，则我们首先要通过 methodSignatureForSelector: 来指定方法签名，返回nil，表示不处理，若返回方法签名，则会进入下一步.
4. 当第 methodSignatureForSelector: 方法返回方法签名后，就会调用 forwardInvocation: 方法，我们可以通过 anInvocation 对象做很多处理，比如修改实现方法，修改响应对象等.
5. 如果到最后，消息还是没有得到响应，程序就会crash.

### 示例：
```objc
#import "Monkey.h"
#import "Bird.h"
#import <objc/runtime.h>

@implementation Monkey

-(void)jump{
    NSLog(@"monkey can not fly, but! monkey can jump");
}

+(BOOL)resolveInstanceMethod:(SEL)sel{
    
    /*
     如果当前对象调用了一个不存在的方法
     Runtime会调用resolveInstanceMethod:来进行动态方法解析
     我们需要用class_addMethod函数完成向特定类添加特定方法实现的操作
     返回NO，则进入下一步forwardingTargetForSelector:
     */
    
	if(sel == @selector(fly)){
    	class_addMethod(self, sel, class_getMethodImplementation(self, sel_registerName("jump")), "v@:");
    	return YES;
    }
    return [super resolveInstanceMethod:sel];
}

-(id)forwardingTargetForSelector:(SEL)aSelector{
    
    /*
     在消息转发机制执行前，Runtime 系统会再给我们一次重定向的机会
     通过重载forwardingTargetForSelector:方法来替换消息的接受者为其他对象
     返回nil则进步下一步forwardInvocation:
     */
    
#if 0
    return nil;
#else
    return [[Bird alloc] init];
#endif
}

-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    
    /*
     获取方法签名进入下一步，进行消息转发
     */
    
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}

-(void)forwardInvocation:(NSInvocation *)anInvocation{
    
    /*
     消息转发
     */
    
    return [anInvocation invokeWithTarget:[[Bird alloc] init]];
}

@end
```
其中 “v@:” 表示返回值和参数.

这一部分看的有点草。

## objc_msgSendSuper
编译器会根据情况在 objc_msgSend，objc_msgSend_stret，objc_msgSendSuper，objc_msgSendSuper_stret 或 objc_msgSend_fpret 五个方法中选择一个来调用。

当我们调用 [super selector] 时，Runtime 会调用 objc_msgSendSuper 方法，objc_msgSendSuper 方法有两个参数，super 和 op，Runtime 会把 selector 方法选择器赋值给 op。而 super 是一个 objc_super 结构体指针，objc_super 结构体定义如下：
```objc
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
```

Runtime 会创建一个 objc_spuer 结构体变量，将其地址作为参数（super）传递给 objc_msgSendSuper，并且将 self 赋值给 receiver：super—>receiver=self.
举个栗子，问下面的代码输出什么：

```objc
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```
答案是全部输出 Son.

当调用 [super class] 时，会转换成 objc_msgSendSuper 函数：

第一步先构造 objc_super 结构体，结构体第一个成员就是 self。第二个成员是 (id)class_getSuperclass(objc_getClass(“Son”)).

第二步是去 Father 这个类里去找 - (Class)class，没有，然后去 NSObject 类去找，找到了。最后内部是使用 objc_msgSend(objc_super->receiver, @selector(class)) 去调用，此时已经和 [self class] 调用相同了，所以两个输出结果都是 Son。

## Runtime-动态创建类添加属性和方法
```objc
- (void)createClass
{
    Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    //添加一个NSString的变量，第四个参数是对其方式，第五个参数是参数类型
    if (class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    }
    //myclasstest是已经实现的函数，"v@:"这种写法见参数类型连接
    class_addMethod(MyClass, @selector(myclasstest:), (IMP)myclasstest, "v@:");
    //注册这个类到runtime系统中就可以使用他了
    objc_registerClassPair(MyClass);
    //生成了一个实例化对象
    id myobj = [[MyClass alloc] init];
    NSString *str = @"asdb";
    //给刚刚添加的变量赋值
    //    object_setInstanceVariable(myobj, "itest", (void *)&str);在ARC下不允许使用
    [myobj setValue:str forKey:@"itest"];
    //调用myclasstest方法，也就是给myobj这个接受者发送myclasstest这个消息
    [myobj myclasstest:10];

}
//这个方法实际上没有被调用,但是必须实现否则不会调用下面的方法
- (void)myclasstest:(int)a
{
    
}
//调用的是这个方法
static void myclasstest(id self, SEL _cmd, int a) //self和_cmd是必须的，在之后可以随意添加其他参数
{
    
    Ivar v = class_getInstanceVariable([self class], "itest");
    //返回名为itest的ivar的变量的值
    id o = object_getIvar(self, v);
    //成功打印出结果
    NSLog(@"%@", o);
    NSLog(@"int a is %d", a);
}
```





