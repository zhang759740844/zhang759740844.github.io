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

## 动态方法解析与消息转发
### 原理
消息转发的大致过程如图：

![消息转发流程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_dynamicMethod.jpg?raw=true)

1. 当 Runtime 系统在`Cache`和方法分发表中（包括超类）找不到要执行的方法时，Runtime会调用`resolveInstanceMethod:`或`resolveClassMethod:`(添加实例方法实现和类方法实现)来给程序员一次动态添加方法实现的机会，指定是否动态添加方法。若返回`NO`，则进入下一步，若返回`YES`，则通过 `class_addMethod` 函数动态地添加方法，消息得到处理，此流程完毕.
2. `resolveInstanceMethod:` 方法返回 `NO` 时，就会进入 `forwardingTargetForSelector:` 方法，这是 Runtime 给我们的第二次机会，用于指定哪个对象响应这个 `selector`。返回`nil`，进入下一步，返回某个对象，则会调用该对象的方法.
3. 若 `forwardingTargetForSelector:` 返回的是`nil`，则我们首先要通过 `methodSignatureForSelector:` 来指定方法签名，返回`nil`，表示不处理，若返回方法签名，则会进入下一步.
4. 当第 `methodSignatureForSelector:` 方法返回方法签名后，就会调用 `forwardInvocation:` 方法，我们可以通过 `anInvocation` 对象做很多处理，比如修改实现方法，修改响应对象等.
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

其中 `"v@:"` 表示返回值和参数,这个符号涉及[Type Encoding](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html)以及[关于type encodings的理解--runtime programming guide](http://www.jianshu.com/p/f4129b5194c0)。

---
其中**v**表示返回`void`类型，**@**表示参数`id(self)`，**：**表示`SEL(_cmd)`这几个是必须要有的，后面可以接入参类型。

举个例子：`"i@:@"`
**i**表示返回值类型`int`
**@：**和上面意义相同
**@**最后一个@表示有一个入参，是`id`类型。

不过，实际上，这个东西好像没什么用，因为在`class_addMethod`上试验过，随便传任何字符串都一样能正常运行。

---
一般来说可以使用`method_getTypeEncoding()`获取更详细的Type_Encoding,下面例子中也会用到。

### 转发与多继承
转发和继承相似，可以用于为Objc编程添加一些多继承的效果。就像下图那样，一个对象把消息转发出去，就好似它把另一个对象中的方法借过来或是“继承”过来一样。
![runtime_transmit](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_transmit.png?raw=true)

这使得不同继承体系分支下的两个类可以“继承”对方的方法，在上图中`Warrior`和`Diplomat`没有继承关系，但是`Warrior`将`negotiate`消息转发给了`Diplomat`后，就好似`Diplomat`是`Warrior`的超类一样。

消息转发弥补了 Objc 不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。它将问题分解得很细，只针对想要借鉴的方法才转发，而且转发机制是透明的。

## Runtime-动态创建类添加属性和方法
```objc
- (void)createClass
{
    Class MyClass = objc_allocateClassPair([NSObject class], "myclass", 0);
    //添加一个NSString的变量，第四个参数是对其方式，第五个参数是参数类型
    if (class_addIvar(MyClass, "itest", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    }
    //myclasstest是已经实现的函数，"v@:"这种写法见type encoding
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
//这个方法实际上没有被调用,但是必须实现否则编译都不能通过
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

## Method Swizzling
此部分参考自[Objective-C的方法替换](http://blog.csdn.net/horkychen/article/details/8532087)、[Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)等系列文章
### 例子
```objc
#import <objc/runtime.h> 
 
@implementation UIViewController (Tracking) 
 
+ (void)load { 
    static dispatch_once_t onceToken; 
    dispatch_once(&onceToken, ^{ 
        Class aClass = [self class]; 
 
        SEL originalSelector = @selector(viewWillAppear:); 
        SEL swizzledSelector = @selector(xxx_viewWillAppear:); 
 
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector); 
        Method swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector); 
        
        // When swizzling a class method, use the following:
        // Class aClass = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(aClass, originalSelector);
        // Method swizzledMethod = class_getClassMethod(aClass, swizzledSelector);
 
        BOOL didAddMethod = 
            class_addMethod(aClass, 
                originalSelector, 
                method_getImplementation(swizzledMethod), 
                method_getTypeEncoding(swizzledMethod)); 
 
        if (didAddMethod) { 
            class_replaceMethod(aClass, 
                swizzledSelector, 
                method_getImplementation(originalMethod), 
                method_getTypeEncoding(originalMethod)); 
        } else { 
            method_exchangeImplementations(originalMethod, swizzledMethod); 
        } 
    }); 
} 
 
#pragma mark - Method Swizzling 
 
- (void)xxx_viewWillAppear:(BOOL)animated { 
    [self xxx_viewWillAppear:animated]; 
    NSLog(@"viewWillAppear: %@", self); 
} 
 
@end
```

### 原理
#### 概述
在Objective-C中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是selector的名字。利用Objective-C的动态特性，可以实现在运行时偷换selector对应的方法实现，达到给方法挂钩的目的。

每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。IMP有点类似函数指针，指向具体的Method实现。如下图
![runtime_swizzling_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_swizzling_1.png?raw=true)

通过Swizzling需要实现的是偷换selector的IMP，如下图所示：
![runtime_swizzling_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_swizzling_2.png?raw=true)

#### 实现过程
上面的代码通过添加一个`Tracking`类别到`UIViewController`类中，将`UIViewController`类的`viewWillAppear:`方法和`Tracking`类别中`xxx_viewWillAppear:`方法的实现相互调换。Swizzling 应该在`+load`方法中实现，因为`+load`是在一个类最开始加载时调用。`dispatch_once`是GCD中的一个方法，它保证了代码块只执行一次，并让其为一个原子操作，线程安全是很重要的。

实现Swizzling需要考虑两种情况，第一种是被替换方法没有在当前类中重写过，而是只在其父类中实现了;第二种情况是这个方法已经存在于当前类中，就如本例中想要替换掉`UIViewController`中的`viewWillAppear:`方法。这两种情况要区别对待。

对于第一种情况，由于当前类内没有这个方法，应当现在当前类中添加一个新的实现方法(`xxx_viewWillAppear:`)，然后将复写的方法替换为原先的实现(`viewWillAppear:`).

```objc
        BOOL didAddMethod = 
            class_addMethod(aClass, 
                originalSelector, 
                method_getImplementation(swizzledMethod), 
                method_getTypeEncoding(swizzledMethod)); 
```
`class_addMethod`将本来不存在于被操作的Class里的`swizzledMethod`的实现添加在被操作的Class里,并使用`originalSelector`作为其选择子。如果发现方法已经存在，会失败返回。

---
通过上一篇[runtime原理](https://zhang759740844.github.io/2016/08/22/runtime原理/)的分析，`class_addMethod`应该是先在类的method数组里找是否有这个`SEL`,如果没有就添加一个`method_t`。

---

如果添加成功(当前类中没有重写过父类的该方法)，再把目标类中的方法替换为旧有的实现:
```objc
        if (didAddMethod) { 
            class_replaceMethod(aClass, 
                swizzledSelector, 
                method_getImplementation(originalMethod), 
                method_getTypeEncoding(originalMethod)); 
```
`addMethod`会让当前类的方法(IMP)指向新的实现(SEL)，使用`replaceMethod`再将新的方法(IMP)指向原先的实现(SEL)，这样就完成了交换操作。现在通过旧方法`SEL`来调用，就会实现新方法的`IMP`，通过新方法的`SEL`来调用，就会实现旧方法的`IMP`。

如果添加失败了，就是第二情况(方法已经在当前类中实现过了)。这时可以通过`method_exchangeImplementations`直接交换两个`method_t`的`IMP`:
```objc
else {
    method_exchangeImplementations(originalMethod, overrideMethod);
}
```

所以本例中由于`viewWillAppear:`已经在UIViewController中实现过了，所以，`class_addMethod`失败，通过`method_exchangeImplementations`达到交换实现。如果要通过`class_addMethod`添加，需要自定义一个View继承UIViewController，再在这个类中替换`viewWillAppear:`。


如果类中没有想被替换实现的原方法时，`class_replaceMethod`相当于直接调用`class_addMethod`向类中添加该方法的实现。

`method_exchangeImplementations`方法做的事情与如下的原子操作等价：
```objc
IMP imp1 = method_getImplementation(m1);
IMP imp2 = method_getImplementation(m2);
method_setImplementation(m1, imp2);
method_setImplementation(m2, imp1);
```
直接设置了`method`:`m1`,`m2`的`IMP`，简单暴力。

对于注释了的这几行:
```objc
// When swizzling a class method, use the following:
// Class aClass = object_getClass((id)self);
// ...
// Method originalMethod = class_getClassMethod(aClass, originalSelector);
// Method swizzledMethod = class_getClassMethod(aClass, swizzledSelector);
```
`object_getClass((id)self)` 与 `[self class]` 返回的结果类型都是 `Class`,但前者为元类,后者为其本身,因为此时 `self` 为 `Class` 而不是实例.注意 `[NSObject class]` 与 `[object实例 class]` 的区别：
```objc
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
```
`object_getClass()`方法返回对象的`isa`。

最后，`xxx_viewWillAppear:`方法的定义看似是递归调用引发死循环，其实不会的。因为`[self xxx_viewWillAppear:animated]`消息会动态找到`xxx_viewWillAppear:`方法的实现，而它的实现已经被我们与`viewWillAppear:`方法实现进行了互换，所以这段代码不仅不会死循环，如果你把`[self xxx_viewWillAppear:animated]换成[self viewWillAppear:animated]`反而会引发死循环。




