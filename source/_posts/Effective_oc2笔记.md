title: Effective Objective-C 学习笔记
date: 2016/11/11 10:07:12  
categories: iOS
tags:
	- Objective-C
---

拜读一下 Effective Objective-C 这本书，做一些笔记

<!--more-->

## 熟悉OC
### 第1条：了解Objective-C的起源

对于消息结构的语言，运行时所执行的代码由运行环境来决定,在运行时才回去查找索要执行的方法;而使用函数调用的语言，则由编译器决定，只有函数是多态的，才会在运行的时候按照“虚方法表”查出到底应该执行哪个函数。

oc 的工作的实现原理是由**运行期组件（runtime component）**完成，而不是编译器。使用 Objective-C 的面向对象特性所需的全部数据结构以及函数都在运行期组件里面。

运行期组件本质上是一种与开发者所编写的代码相链接的**动态库（dynamic library）**，其代码能把开发者所编写的所有程序粘合起来。这样的话，只要更新运行期组件，就可以提升程序性能。而那种工作都在 “编译期” 完成的语言，若想获得类似的性能提升，就要重新编译应用程序代码。

### 第2条： 在类的头文件中尽量少引用其他头文件

有时，类A需要将类B的实例变量作为它公共 API 的属性。这个时候，我们不应该引入类B的头文件，而应该使用**向前声明（forward declaring）** 使用 `@class` 关键字，并且**在 A 的实现文件引用 B 的头文件**。(继承或者协议必须引入完整头文件，不能使用向前声明)

```objc
// EOCPerson.h
#import <Foundation/Foundation.h>

@class EOCEmployer;

@interface EOCPerson : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, strong) EOCEmployer *employer;//将EOCEmployer作为属性

@end

// EOCPerson.m
#import "EOCEmployer.h"
```

这样做有什么优点呢：
1. 不在A的头文件中引入B的头文件，那么A的实现文件引入A的头文件时，就不会一并引入B的全部内容，这样就减少了编译时间。只有在A的实现里需要用到B时，再在A的实现里引用B的头文件。
2. 使用 `#import` 而不是 `#include` 可以避免死循环，但仍会导致相互引用的两个类中的一哥无法正确编译。使用 `@class` 可以避免循环引用：因为如果两个类在自己的头文件中都引入了对方的头文件，那么就会导致其中一个类无法被正确编译。

### 第3条：多用字面量语法，少用与之等价的方法
声明时多用字面量：

```objc
NSNumber *intNumber = @1;
NSNumber *floatNumber = @2.5f;
NSArray *animals =@[@"cat", @"dog",@"mouse", @"badger"];
Dictionary *dict = @{@"animal":@"tiger",@"phone":@"iPhone 6"};

NSString *cat = animals[0];
NSString *iphone = dict[@"phone"];
```

少用 `alloc`、`init` 的方式创建，以及 `objectAtIndex`、`objectForKey` 的方式取数组字典。

优点：
1. 简洁
2. `NSArray` 以 `nil` 结尾，所以一般不允许数组中的元素为 `nil`，如果使用等价方法，那么数组元素为 `nil` 不报错，会出现难以排查的错误；而同样的情况，字面量语法会抛出异常。

### 第4条：多用类型常量，少用#define预处理命令
预处理与类型常量的优缺点：
- 预处理命令：简单的文本替换，不包括类型信息，并且可被任意修改。
- 类型常量：包括类型信息，并且可以设置其使用范围，而且不可被修改。

#### 预处理命令

```objc
#define W_LABEL (W_SCREEN - 2*GAP)
```

这里，`(W_SCREEN - 2*GAP)` 替换了 `W_LABEL`，它不具备 `W_LABEL` 的类型信息。而且要注意一下：如果替换式中存在运算符号，以笔者的经验最好用括号括起来，不然容易出现错误（有体会）。

#### 类型常量

```objc
static const NSTimeIntervalDuration = 0.3;
```

`const` 将其设置为常量，不可更改。`static` 意味着该变量仅仅在定义此变量的编译单元(`.m` 实现文件)中可见。如果不声明 `static`,编译器会为它创建一个**外部符号（external symbol）**。会出现什么问题呢？如果在其他类中也声明了同名变量，即使没有相互引用，编译器也会抛出一个异常。

#### 全局常量
如果我们需要发送通知，那么就需要在不同的地方拿到通知的“频道”字符串，那么显然这个字符串是不能被轻易更改，而且可以在不同的地方获取。这个时候就需要定义一个外界可见的字符串常量，即全局常量。在头文件中声明外部常量，在实现文件中完成变量的赋值。

```objc
//header file
extern NSString *const NotificationString;

//implementation file
NSString *const  NotificationString = @"Finish Download";
```

注意这里的 `const`, 如果在 `*` 前面，表示指针指向的堆上的内容不能改变，如果在 `*` 后面，表示指针指向的地址是不能改变的。（这里有个助记方法，以 `*` 为分解，`const` 在左边就是修饰 `NSString`，表示不能修改值，在右边就表示修饰指针对象，表示不能修改指针指向的地址。）

> 不要用预处理指令定义**常量**。这样定义出来的常量不含类型信息，编译器只会在百年以前执行查找和替换操作。即使有人重新定义了常量值，编译器也不会产生警告信息，这回导致应用程序中的常量值不一致。

### 第5条：用枚举表示状态，选项，状态码
我们经常需要给类定义几个状态，这些状态码可以用枚举来管理：

```objc
typedef NS_ENUM(NSUInteger, EOCConnectionState) {
  EOCConnectionStateDisconnected,
  EOCConnectionStateConnecting,
  EOCConnectionStateConnected,
};

typedef NS_OPTION(NSUInteger, EOCPermittedDirection) {
	EOCPermittedDirectionUp    = 1 << 0,
	EOCPermittedDirectionDown  = 1 << 1,
	EOCPermittedDirectionLeft  = 1 << 2,
	EOCPermittedDirectionRight = 1 << 3
}
```

`NS_ENUM` 和 `NS_OPTION` 是 Foundation 框架中定义的辅助宏。需要注意这两者使用场景的不同。

> 这两者的差别在于一个位移枚举即是在你需要的地方可以同时存在多个枚举值。而NS_ENUM定义的枚举不能几个枚举项同时存在，只能选择其中一项。

在枚举类型的 switch 语句中不要实现 default 分支。它的好处是，当我们给枚举增加成员时，编译器就会提示开发者：switch 语句并未处理所有的枚举。否则添加了枚举却没有实现 switch 将可能导致严重的崩溃。

注意，switch 的 `case`中如果声明了变量，必须要用`{}`包住，这是编译器强制的，不然会报错，例子：

```objc
- (void)startAnimationInitialWithType:(NSInteger)type{
    switch (type) {
        case BaseAnimation:{
            CALayer *myLayer=[CALayer layer];

            //添加layer
            [self.view.layer addSublayer:myLayer];
            self.myLayer=myLayer;
        }
            break;
        default:
            break;
    }
}
```



## 对象、消息、运行期
### 第6条：理解“属性”这一概念
#### 属性
在 Java 以及 C++ 中，对象布局在编译期就已经固定了。只要访问变量的代码，编译器就会把其替换成为“偏移量”。这个偏移量是**硬编码（hardcode）**，表示该对象距离存放对象的内存区域的起始地址有多远。这样做的问题是，如果再添加一个实例变量，那么其他实例变量的就要变化了，那么就要重新编译，否则就会出错。

Objective-C 的做法是，把实例变量当做一种存储偏移量所用的**“特殊变量”（special variable）**，交由**“类对象”（class object）**保管。偏移量会在运行期查找，那么类的定义变了，存储的偏移量也就变了，这样的话，无论何时访问实例变量，总能使用正确的偏移量。甚至可以在运行期向类中新增实例变量，这就是稳固的 "应用程序二进制接口(ABI)"。ABI 定义了许多内容，其中一项就是生成代码时所应遵循的规范(这也就是 swift 所没有的东西)。

#### 存取方法
**在设置完属性后，编译器会自动向类中添加适当类型的实例变量，并且为其写出一套存取方法。**一般会在属性名前加一个下划线作为实例变量名。

如果不想令编辑器自动合成存取方法，可以自己实现，也可以使用 `@dynamic` 关键字。它会告诉编译器不要自动创建实现属性所用的实例变量，也不要为其创建存取方法，**需要自己实现存取方法**。而且，在编译访问属性的代码时，即使编译器发现没有定义存取方法，也不会报错，它相信这些方法能在运行期找到。

访问属性，可以使用点语法，编译器会把点语法转换为对存取方法的调用；也可直接使用实例变量，使用实例变量的方式更快。

```objc
//存取方法设置属性
self.firstName = @"Zachary";
//实例变量设置属性
_firstName = @"Zachary";
```

#### @synthesize 与 @dynamic

`@dynamic` 是相对于 `@synthesize` 的，它们用样用于修饰 `@property`：

```objc
//.h
@interface CYLPerson : NSObject 
@property NSString *firstName; 
@property NSString *lastName; 
@end
  
//.m
@implementation CYLPerson 
@synthesize firstName = _myFirstName; 
@synthesize lastName = _myLastName; 
@end 
```

上述语法会将生成的实例变量（ivar）命名为 `_myFirstName` 与 `_myLastName`，并自动合成 `setFirstName:` 和 `firstName`,`setLastName`,`lastName` 这几个方法。

如果是 `@synthesize foo;`，等效于 `@synthesize foo = foo`，省略了下划线，后面使用该实例变量的时候就用 `foo` 就行了。如果已经存在一个名为 `_foo` 的实例变量，就不会自动合成新的变量了：

![foo](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/effectiveoc_foo.png?raw=true)

`@dynamic` 类似，告诉编译器，不自动生成getter/setter方法，避免编译期间产生警告，然后由自己实现存取方法或在运行时动态绑定。

> `@synthesize` (Xcode6以后省略这个了, 默认在 `@implementation .m` 中添加这个 `@synthesize xxx = _xxx;` )

#### 属相特质

原子性：
- nonatomic：不使用同步锁
- atomic：加同步锁，确保其原子性

读写:
- readwrite:同时存在存取方法
- readonly:只有获取方法

内存管理:
- assign:纯量类型(scalar type)的简单赋值操作
- strong:拥有关系保留新值，释放旧值，再设置新值
- weak:非拥有关系(nonowning relationship)，属性所指的对象遭到摧毁时，属性也会清空
- copy：当赋给其可变对象，返回不可变对象；当赋给其不可变对象，返回原对象。

> 块要用 copy 最好不要用 strong。
>
> 可能出现的 retain 关键字一般情况下等同于 strong 

#### weak的实现

这里插一条 `weak` 是如何实现的。一般内存是通过 ARC 管理的。使用 `weak` 不增加对象的引用次数。当栈中的变量不指向堆中的对象时，堆中对象销毁。这个时候要把 `weak` 指向的地址置为 nil，因为如果不这么做，那么就会产生野指针。那么这是如何做到的？

runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak 表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。

### 第7条： 在对象内部尽量直接访问实例变量
关于实例变量的访问，可以直接访问 `_firstName`，也可以通过属性的方式(点语法) `self.firstName` 来访问。书中作者建议在读取实例变量时采用直接访问的形式，而在设置实例变量的时候通过属性来做。（这部分比较重要）

直接访问实例变量的特点：
- 不经过**“方法派发”(method dispatch)**，会直接访问保存对象实例变量的那块内存，速度快。

通过属性访问实例变量的特点：
- 不会绕过属性定义的**内存管理语义**。其实也就是说，编译期在设置 set 方法的时候，会根据属性特质做一些操作。比如一个声明为 `copy` 的属性，如果直接访问实例变量，那么这个实例变量就会直接指向堆中的对象；而如果通过属性来操作，就会先将堆中的对象 copy 一份，然后将实例变量指向 copy 出来的对象。
- 可以触发KVO( KVO 是通过 aop 在设置方法中加的通知 )

不过有两个特例：
1. `init` 方法和 `dealloc` 方法中，需要直接访问实例变量来进行设置属性操作。因为如果在这里没有绕过set方法，就有可能触发其他不必要的操作(比如上面说的**内存管理语义**所要进行的操作)。
2. 如果使用**懒加载**的获取方法要用属性的方式获取。

其实，到底用什么很简单，如果 get，set 方法里没有其他的乱七八糟的东西，比如:

```objc
- (NSString *)firstName{
	return _firstName;
}

- (void) setFirstName:(NSString *)firstName{
	_firstName = firstName;
}
```

上面这种，那就直接用实例变量操作了，用属性就是多此一举；如果有乱七八糟的东西，那么就要用属性的方式。

### 第8条：理解“对象等同性”这一概念
`NSObject` 类中有两个用于判断等同性的方法：
- `- (BOOL)isEqual:(id)object;`
- `- (NSUInteger)hash;`

`NSObject` 类中默认的实现是：当且仅当其内存地址完全相等时，两个对象才相等。自定义对象中可以覆写这两个方法（其实好像没必要重写 hash 方法，因为我们重写的 `isEqual:` 方法里根本没有用到 hash 方法，重写了也没啥用），完成自己的相等判断。如果 `isEqual:` 方法判断对象相等，那么其 hash 方法也必须返回同一个值；反之，如果 hash 方法返回了同一个值，`isEqual:` 方法未必认为两者相等。

如果已知两个对象是字符串，最好通过 `isEqualToString:` 方法来比较。对于数组和字典，也有 `isEqualToArray:` 方法和 `isEqualToDictionary:`方法。

如果比较的对象类型和当前对象类型相同，就可以采用自己编写的判定方法，否则调用父类的 `isEqual:` 方法：
```objc
- (BOOL)isEqualToPerson:(EOCPerson*)otherPerson {

     //先比较对象类型，然后比较每个属性
     if (self == object) return YES;
     if (![_firstName isEqualToString:otherPerson.firstName])
         return NO;
     if (![_lastName isEqualToString:otherPerson.lastName])
         return NO;
     if (_age != otherPerson.age)
         return NO;
     return YES;
}


- (BOOL)isEqual:(id)object {
    //如果对象所属类型相同，就调用自己编写的判定方法，如果不同，调用父类的isEqual:方法
     if ([self class] == [object class]) {    
         return [self isEqualToPerson:(EOCPerson*)object];
    } else {    
         return [super isEqual:object];
    }
}
```



### 第9条 以“类族模式“隐藏实现细节

**其实就是通过抽象类完成工厂模式。**

例如,对于“员工”这个类，可以有各种不同的“子类型”：开发员工，设计员工和财政员工。这些“实体类”可以由“员工”这个抽象基类来获得：

```objc
//EOCEmployee.h

typedef NS_ENUM(NSUInteger, EOCEmployeeType) {
    EOCEmployeeTypeDeveloper,
    EOCEmployeeTypeDesigner,
    EOCEmployeeTypeFinance,
};

@interface EOCEmployee : NSObject

@property (copy) NSString *name;
@property NSUInteger salary;


// Helper for creating Employee objects
+ (EOCEmployee*)employeeWithType:(EOCEmployeeType)type;

// Make Employees do their respective day's work
- (void)doADaysWork;

@end
```

```objc
//EOCEmployee.m

@implementation EOCEmployee

+ (EOCEmployee*)employeeWithType:(EOCEmployeeType)type {
     switch (type) {
         case EOCEmployeeTypeDeveloper:
            return [EOCEmployeeDeveloper new];
         break; 

        case EOCEmployeeTypeDesigner:
             return [EOCEmployeeDesigner new];
         break;

        case EOCEmployeeTypeFinance:
             return [EOCEmployeeFinance new];
         break;
    }
}

- (void)doADaysWork {
 // 需要子类来实现
}

@end
```

```objc
@interface EOCEmployeeDeveloper : EOCEmployee
@end

@implementation EOCEmployeeDeveloper

- (void)doADaysWork {
    [self writeCode];
}

@end
```

这样，表面上对象是 `EOCEmployee`，但是实际上操作的是 `EOCEmployeeDeveloper`。

这里需要注意一点：对于这种类族，不能通过以下方式判断：

```objc
if ([employeeDeveloper class] == [EOCEmployee class]){
	// will do
}
```

因为 `employeeDeveloper` 对象是 `EOCEmployee` 类的一个子集，需要使用 `isKindOfClass:` 方法：

```objc
if ([employeeDeveloper isKindOfClass:[EOCEmployee class]]){
	// will do
}
```

### 第10条：在既有类中使用关联对象存放自定义数据

这一条和 runtime 息息相关。背景是，我们可以通过 category 为系统类添加方法，但是无法添加属性。当需要为系统类添加属性时，可以使用下面的方法：

```objc
//为某个对象设置关联对象的值，第一个参数是主对象，第二个参数是键，第三个参数是关联的对象，第四个参数是存储策略:是枚举，定义了内存管理语义
void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)

//根据给定的键从某对象中获取相应的关联对象值
id objc_getAssociatedObject(id object, void *key)

//移除指定对象的关联对象
void objc_removeAssociatedObjects(id object)
```

对象关联类型 `objec_AssociationPolicy` 包括：

```objc
OBJC_ASSOCIATION_ASSIGN				//assign
OBJC_ASSOCIATION_RETAIN_NONATOMIC	//nonatomic,retain
OBJC_ASSOCIATION_COPY_NONATOMIC		//nonatomic,copy
OBJC_ASSOCIATION_RETAIN				//retain	
OBJC_ASSOCIATION_COPY				//copy
```



这里要强调的是，要拿到设置的属性，键必须要完全相等。因此，需要设置成静态全局变量：

```objc
static void *EOCMyAlertViewKey = "EOCMyAlertViewKey";
```


### 第11条：理解objc_msgSend的作用
这部分包括下面几个在[runtime](https://zhang759740844.github.io/2016/08/22/runtime原理/)中已经写得很详细了。

### 第12条：理解消息转发机制
### 第13条：用“方法调配技术”调试“黑盒方法”
### 第14条：理解“类对象”的用意
## 接口与API设计
### 第15条：用前缀 避免命名空间冲突
Apple 宣称其保留使用所有"两字母前缀"的权利，所以我们选用的前缀应该是三个字母的。而且，如果自己开发的程序使用到了第三方库，也应该加上前缀。

自己开发的程序库用到了第三方库，则应为其中的名称加上前缀。(用 cocoapods 可以自动加上前缀，自己开发的库的话要手动改名。)

### 第16条：提供"全能初始化方法"
所谓全能初始化方法，就是所有初始化方法都要调用的初始化方法。这个初始化方法初始化方法是初始化方法里参数最多的一个，因为它使用了尽可能多的初始化所需要的参数，以便其他的方法来调用自己。

算是一种写代码的技巧吧。平时写代码的时候也都是这样的，不具体说明了。

### 第17条：实现description方法
自定义的类调用 `NSLog();` 的时候，往往不能返回想要的结果。需要重写 `NSObject` 类中的 `description` 方法，返回需要的字符串。
```objc
- (NSString*)description {
     return [NSString stringWithFormat:@"<%@: %p, %@ %@>", [self class], self, firstName, lastName];
}
```

其中，`%p` 表示对象的内存地址。

### 第18条：尽量使用不可变对象
尽量使用不可变对象。没啥可说的。
里面推荐的方法没用过，感觉并不好，就不写了。

### 第19条：使用清晰而协调的命名方式
没啥好说的，注意就好

### 第20条：为私有方法名加前缀
建议在实现文件里将非公开的方法都加上前缀，便于调试，而且这样一来也很容易区分哪些是公共方法，哪些是私有方法。因为往往公共方法是不便于任意修改的。

```objc
#import <Foundation/Foundation.h>

@interface EOCObject : NSObject

- (void)publicMethod;

@end


@implementation EOCObject

- (void)publicMethod {
 /* ... */
}

- (void)p_privateMethod {
 /* ... */
}

@end
```

很有用的建议，就像上面一样在私有方法前面加上 `p_` 挺好的。注意不要**单用**下划线来区分私有方法和公共方法，因为会和苹果公司的API重复。

### 第21条：理解Objective-C错误类型
OC 中仅在及其严重的错误情况下抛出异常。比如一个抽象基类。由于 OC 中没办法将某个类标识为抽象类。如果想要实现抽象类的功能，那么就要在必须要覆写的方法里抛出异常：

```objc
- (void)mustOverrideMethod{
	NSString *reason = [NSString stringWithFormat:@"%@ must be overridden", NSStringFromeSelector(_cmd)];
	@throw [NSException exceptionWithName:NSInternalInconsistencyException
									    reason:reason
									  userInfo:nil];
}
```

对于不严重的异常，可以使用回调块的方式返回 `nil` 或者 `NSError` 抛给方法的调用者处理，比如各种网络库都是这么做的，输入一个成功回调，和一个失败回调。

### 第22条：理解NSCopying协议
#### 自定义拷贝
如果我们想令自己的类支持拷贝操作，那就要实现 `NSCopying` 协议，该协议只有一个方法：

```objc
- (id)copyWithZone:(NSZone*)zone
```

比如要拷贝一个 `EOCPerson` 对象：

```objc
- (id)copyWithZone:(NSZone*)zone {
     EOCPerson *copy = [[[self class] allocWithZone:zone] initWithFirstName:_firstName  andLastName:_lastName];
    copy->_friends = [_friends mutableCopy];
     return copy;
}
```



这里面的 `NSZone *zone` 对象是以前开发程序时，会根据此吧内存分成不同的区(zone)，对象会被创建在某个区里面。现在不用了，每个程序都只有一个区："默认区"，所以现在不必在意这个对象。这个方法就是新建一个 `EOCPerson` 对象，然后调用它的构造函数把东西全都塞进去。这里的 `->` 用箭头是因为定义的时候这个 `_friends` 不是一个属性(代码没有贴出来，详见书)，而只是在实现文件中定义的一个实例变量，没有 get/set 方法，所以不能用 `.`，一般情况用点语法就行了。

这里的 `mutableCopy` 方法也可以自定义，就是下面方法的实现：

```objc
-(id)mutableCopyWithZone:(NSZone*)zone；
```



#### 浅拷贝与深拷贝

浅拷贝和深拷贝应该并不陌生。浅拷贝只增加引用计数，深拷贝将创建另一个一模一样的对象。

- 不可变对象的 `copy` 是浅拷贝。
- 可变对象的 `copy` 是深拷贝，返回不可变对象。
- 不可变对象的 `mutableCopy` 是深拷贝，返回可变对象。
- 可变对象的 `mutableCopy` 是深拷贝。

容器对象(`NSArray`)本身也遵循上面的规则。但是需要注意的是，**容器对象内的元素是浅拷贝（你可能通过深拷贝新建了一个 NSArray，但是 NSArray 里面存的还是对象的指针，还是可以修改 NSArray 里的对象的，两个 NSArray 里的对象都会被修改）**。因此上面的自定义 copy 方法如果想让 `_friends` 内的元素深拷贝，就不能用 `[_friends mutableCopy]` 方法，需要新建一个 Set:

```objc
- (id)deepCopy {
   EOCPerson *copy = [[[self class] alloc] initWithFirstName:_firstName andLastName:_lastName];
	copy->_friends = [[NSMutableSet alloc] initWithSet:_friends copyItems:YES];
   return copy;
}
```

## 协议与分类
### 第23条：通过委托与数据源协议进行对象间通信
其实也是老生常谈的东西了，不过也有一些注意点。

受代理对象内持有代理对象的实例时要写成这样：

```objc
@property (nonatomic, weak) id <NetworkDelegate> delegate;
```

这里书中指明了要用 `weak`，不能用 `strong`，否则会引起引用循环。比如系统中的 `TableViewCellDelegate`，在 `TableViewController` 作为代理类确实拥有被代理对象 `TableViewCell`，用 `weak` 确实是合理的，但是所有情况都这样吗？不知道，不过确实基本上的代理对象都是 `ViewController`，所以用 `weak` 肯定是不会有问题的。

实现委托对象的方法是声明某个类遵从委托协议：

```objc
@implementation EOCDataModel () <EOCNetworkFetcherDelegate>
@end
@implementation EOCDataModel
// 各个实现方法
@end
```

基本所有的 Delegate 都在 `.m` 文件中的类拓展中声明，之前一直没有留意，看了书后才问自己，为什么不在 `.h` 中声明？两者有什么差别吗？其实也没什么差别，在实现文件中声明的好处是能隐藏细节。如果只是自己用可能没什么区别，但是如果打包给别人用，那么就不应该让别人看到你的实现细节了，因此，就把这个 Delegate 的声明放到了实现文件中。


### 第24条：将类的实现代码分散到便于管理的数个分类中
当一个类越来越大时，就变得不利于管理，因此需要将类代码按照逻辑划分入几个分区中，可以通过范畴的方式实现。书中有一个例子：

无分类:
```objc
#import <Foundation/Foundation.h>

@interface EOCPerson : NSObject

@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
@property (nonatomic, strong, readonly) NSArray *friends;

- (id)initWithFirstName:(NSString*)firstName andLastName:(NSString*)lastName;

/* Friendship methods */
- (void)addFriend:(EOCPerson*)person;
- (void)removeFriend:(EOCPerson*)person;
- (BOOL)isFriendsWith:(EOCPerson*)person;


/* Work methods */
- (void)performDaysWork;
- (void)takeVacationFromWork;


/* Play methods */
- (void)goToTheCinema;
- (void)goToSportsGame;


@end 
```
分类后:
```objc

#import <Foundation/Foundation.h>


@interface EOCPerson : NSObject

@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
@property (nonatomic, strong, readonly) NSArray *friends;



- (id)initWithFirstName:(NSString*)firstName

andLastName:(NSString*)lastName;

@end



@interface EOCPerson (Friendship)

- (void)addFriend:(EOCPerson*)person;
- (void)removeFriend:(EOCPerson*)person;
- (BOOL)isFriendsWith:(EOCPerson*)person;

@end



@interface EOCPerson (Work)

- (void)performDaysWork;
- (void)takeVacationFromWork;

@end



@interface EOCPerson (Play)

- (void)goToTheCinema;
- (void)goToSportsGame;

@end
```

如果觉得写在一个实现文件中太长了，可以拆开，比如将其中的 `Friendship` 拆开。

```objc
// EOCPerson+Friendship.h
#import "EOCPerson.h"


@interface EOCPerson (Friendship)

- (void)addFriend:(EOCPerson*)person;
- (void)removeFriend:(EOCPerson*)person;
- (BOOL)isFriendsWith:(EOCPerson*)person;

@end


// EOCPerson+Friendship.m
#import "EOCPerson+Friendship.h"


@implementation EOCPerson (Friendship)

- (void)addFriend:(EOCPerson*)person {
 /* ... */
}

- (void)removeFriend:(EOCPerson*)person {
 /* ... */
}

- (BOOL)isFriendsWith:(EOCPerson*)person {
 /* ... */
}

@end
```

不过要注意，在新建分类文件时，一定要引入被分类的类文件。

**这是一个很有用的将大文件拆分的技巧啊**。

> 分类和协议都可以用 @property 定义属性，但是都只是声明了方法，没有定义成员变量



### 第25条：总是为第三方类的分类名称加前缀

如果我们想给第三方库或者iOS框架里的类添加分类时，最好将分类名和方法名加上前缀。否则可能会替换掉系统的方法。

### 第26条:勿在分类中声明属性
这本书要是早点看到就好了，当时纠结这个很久，走了很多弯路。

分类机制，目标在于扩展类的功能，而不是封装数据。

### 第27条：使用class-continuation分类 隐藏实现细节
通常，我们需要减少在公共接口中向外暴露的部分(包括属性和方法)，而因此带给我们的局限性可以利用 class-continuation 分类的特性来补偿：
- 可以在 class-continuation 分类中增加实例变量。
- 可以在 class-continuation 分类中将公共接口的只读属性设置为读写。(这个看起来也挺有用的，外部无法修改，内部却能更改)
- 可以在 class-continuation 分类中遵循协议，使其不为人知。

例子一：

```objc
@interface EOCPerson()<EOCPersonDelegate>
// method
@end
```

例子二：

```objc
//.h
@interface EOCPerson:NSObject
@property (nonatomic,copy,readonly) NSString *firstName;
@property (nonatomic,copy,readonly) NSString *lastName;

- (id)initWithFirstName:(NSString *)firstName lastName:(NSString *)lastName;

//.m
@interface EOCPerson()
@property (nonatomic,copy,readwrite) NSString *firstName;
@property (nonatomic,copy,readwrite) NSString *lastName;
```



### 第28条:通过协议提供匿名对象
OC 里的**匿名对象**和 Java 里的匿名对象不同，这里的匿名对象没有类型。有时我们用协议来提供匿名对象，目的在于说明它仅仅表示“遵从某个协议的对象”，而不是“属于某个类的对象”。它的表示方法为：`id<protocol>`。

通过协议提供匿名对象的主要使用场景有两个：
- 作为属性
- 作为方法参数

#### 匿名对象作为属性
在设定某个类为自己的代理属性时，可以不声明代理的类，而是用 `id<protocol>`，因为成为代理的终点并不是某个类的实例，而是遵循了某个协议。

```objc
@property (nonatomic, weak) id <EOCDelegate> delegate;
```

在这里使用匿名对象的原因有两个：
1. 将来可能会有很多不同类的实例对象作为该类的代理。
2. 我们不想指明具体要使用哪个类来作为这个类的代理。


也就是说，能作为该类的代理的条件只有一个：它遵从了 `<EOCDelegate>` 协议。

#### 匿名对象作为方法参数
有时，我们不会在意方法里某个参数的具体类型，而是遵循了某种协议，这个时候就可以使用匿名对象来作为方法参数。

```objc
- (void)setObject:(id)object forKey:(id<NSCopying>)key;
```

这个方法是 NSDictionary 的设值方法，它的参数只要遵从了 `<NSCopying>` 协议，就可以作为参数传进去,作为 NSDictionary 的键。

## 内存管理
### 第29条：理解引用计数
`NSObject` 协议声明了下面三个方法用于操作计数器，以递增或递减其值：

- retain: 递增保留计数
- release： 递减保留计数
- autorelease： 待稍后清理“自动释放池”时，再递减保留计数。

对象创建出来时，其保留计数至少为1。若想令其继续存活，则调用 `retaion` 方法。要是某部分代码不在使用该对象，则调用 `release` 或 `autorelease`。最终当保留计数归零时，对象就回收了(dealloced)。

如果按照引用树回溯，那么最终会发现一个根对象。在 iOS 中是 `UIApplication` 。两者都是应用程序启动时创建的单例。

调用 `autorelease` 会在稍后递减计数，通常是下一个事件循环。这个特性可以在方法返回对象时用到：

```objc
- (NSString *)stringValue{
  NSString *str = [[NSString alloc] initWithFormat:@"I am this: %@",self];
  return [str autorelease];
}
```

在 `alloc` 的时候，引用计数加一，返回的时候要将这次引用抵消，所以使用 `autorelease`。修改后，`stringValue` 方法把 `NSString` 对象返回给调用者的时候，此对象必然存活。所以我们能用下面这样使用：

```objc
NSString *str = [self stringValue];
NSLog(@"The string is: %@",str);
```

由于返回的 `str` 将于稍后自动释放，所以多出来的那一次保留操作到时候会自然抵消，无须执行任何内存管理操作。因为自动释放池中的释放操作要等到下一个事件循环才能执行，所以 `NSLog` 语句在使用 `str` 对象前不需要手动执行保留操作。但是如果要持有此对象的话，那就需要保留，然后手动释放了：

```objc
_instanceVariable = [[self stringValue] retain];
//...
[_instanceVariable release];
```

由此可见，`autorelease` 可以延长对象生命期，使其在跨越方法调用边界后依然可以存活一段时间。

### 第30条：以ARC简化引用计数
引用计数还是要执行的，只不过保留和释放操作现在由 ARC 自动添加，可以省略对于引用计数的操作。由于 ARC 会执行 `retain` `release` `autorelease` `dealloc`，所以直接调用这些方法是非法的。

需要了解一个修饰符 `__weak`。块内引用外部变量时，会自动保留其所捕获的全部对象，如果这其中有某个对象保留了块本身（如将 ViewController 传入），将会形成“保留环”。所以要用 `__weak` 局部变量来打破这种保留环。

```objc
EOCNetwork * __weak weakFetcher = fetcher;
```

我们在使用 block 的过程中，经常会需要引用 `self`，为了打破引用循环，我们需要这么做：

```objc
// block 外
__weak typeof(self) wself = self;

// block 内
// 先判断 self 是否已被回收，然后再强引用 wself，使之不会在 block 执行的时候被回收
if (!wself) return;
__strong typeof(wself) sself = wself
```



### 第31条：在dealloc方法中只释放引用并解除监听
对象在经历生命期后，最终会被系统回收，这里就是执行 `dealloc` 方法了。永远不要自己调用 `dealloc` 方法，运行期系统会在适当的时候调用它。根据性能需求我们有时需要在 `dealloc` 方法中做一些操作。那么我们可以在 `dealloc` 方法里做什么呢？
- 释放对象所拥有的所有引用，不过ARC会自动添加这些释放代码，可以不必操心。
- 对象拥有的其他非OC对象也要释放（CoreFoundation 对象就必须手动释放）
- 释放原来的观测行为：注销通知。如果没有及时注销，就会向其发送通知，使得程序崩溃。

例如：
```objc
- (void)dealloc {
     CFRelease(coreFoundationObject);
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```

除了释放引用和注销通知，不要在 `dealloc` 中做其他任何事（比如调用属性的存取方法，以及异步操作）。如果对象持有文件描述符等系统资源，那么应该专门编写一个方法来释放该资源。这样的类要和使用者约定，用完资源后必须调用 `close` 方法。

### 第32条：编写“异常安全代码”时留意内存管理问题
在发生异常时的内存管理需要仔细考虑内存管理的问题：在 try 块中，如果先保留了某个对象，然后在释放它之前又抛出了异常，那么除非在 catch 块中能处理此问题，否则对象所占内存就将泄漏。

```objc
@try {
     EOCSomeClass *object = [[EOCSomeClass alloc] init];
     [object doSomethingThatMayThrow];
}
@catch (...) {
 NSLog(@"Whoops, there was an error. Oh well...");
}
```

所以一定要注意将 `try` 块内所创建的对象处理干净。

### 第33条：以弱引用避免保留环
对象之间都用强指针引用对方的话会造成保留环。如果保留环连接了多个对象，而这里其中一个对象被外界引用，那么当这个引用被移除后，整个保留环就泄漏了。不像 Java 那种处理方式，OC 中孤立的保留环不能被自动释放。

![保留环](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/effective_oc1.png?raw=true)


那么就要用弱引用的方式：

```objc
//EOCClassB.m
//第一种弱引用：unsafe_unretained
@property (nonatomic, unsafe_unretained) EOCClassA *other;


//第二种弱引用：weak
@property (nonatomic, weak) EOCClassA *other;
```

这两种弱引用有什么区别呢？
当指向 `EOCClassA` 实例的引用移除后，`unsafe_unretained` 属性仍然指向那个已经回收的实例，而 `weak` 指向 `nil`。显然，用 `weak` 字段应该是更安全的，因为不再使用的对象按理说应该设置为 `nil`,而不应该产生依赖。

所以只要用 `weak` 就行了。

### 第34条：以“自动释放池快”降低内存峰值
这个部分在 `runloop` 相关文章中有学习过。主要用的是这样一个例子：

```objc
for (int i = 0; i < 100000; i++) {
      [self doSomethingWithInt:i];
}
```

由于线程自动释放池在 event loop 时，进行清空，上面的代码将可能造成内存峰值。因此，可以手动添加一个自动释放池，把循环内的代码包裹在内，那么循环中自动释放的独享就会在这个池中，而不是在线程的主池中。

```objc
NSArray *databaseRecords = /* ... */;
NSMutableArray *people = [NSMutableArray new];
for (NSDictionary *record in databaseRecords) {
     @autoreleasepool {
             EOCPerson *person = [[EOCPerson alloc] initWithRecord:record];
            [people addObject:person];
      }
}
```


### 第35条：用“僵尸对象”调试内存管理问题

某个对象被回收后，再向它发送消息是不安全的，这并不一定会引起程序崩溃。如果程序没有崩溃，可能是因为：

- 该内存的部分原数据没有被覆写。
- 该内存恰好被另一个对象占据，而这个对象可以应答这个方法。

如果被回收的对象占用的原内存被新的对象占据，那么收到消息的对象就不会是我们预想的那个对象。在这样的情况下，如果这个对象无法响应那个方法的话，程序依旧会崩溃。因此，我们希望可以通过一种方法捕捉到对象被释放后收到消息的情况。这种方法就是利用僵尸对象！

Cocoa 提供了“僵尸对象”的功能。如果开启了这个功能，运行期系统会把所有已经回收的实例转化成特殊的“僵尸对象”（通过修改 isa 指针，令其指向特殊的僵尸类），而不会真正回收它们，而且它们所占据的核心内存将无法被重用，这样也就避免了覆写的情况。在僵尸对象收到消息后，会抛出异常，它会说明发送过来的消息，也会描述回收之前的那个对象。

(感觉好像用处不太大的样子。)
### 第36条：不要使用retainCount
ARC 后，这个 `retainCount` 方法就废弃了。反正从来没用过，也就没啥好看的了。

## 块与大小枢派发
### 第37条：理解“块”这一概念
基本概念无需多说，这里强调一下块的种类。

块分为三类：
- 栈块
- 堆块
- 全局块

#### 栈块
这是比较容易被忽略的一块。定义块的时候，其所占内存区域是**分配在栈中**的，而且只在定义它的那个范围内有效：

```objc
void (^block)();

if ( /* some condition */ ) {
    block = ^{
     NSLog(@"Block A");
    };

} else {
    block = ^{
     NSLog(@"Block B");
    };
}

block();
```

上面定义的两个块只在 `if else` 语句范围内有效，一旦离开了最后一个右括号，如果编译器覆写了分配给块的内存，那么就会造成程序崩溃。

并不明白把块保存在栈上是个什么机制。应该可以这么理解吧:`block` 是一个指向栈上内存的指针，栈和堆的引用机制不同，在代码块运行结束后就会将代码块中的局部变量出栈，这个时候 `block` 指向的地方就被回收，`block` 就成了野指针，因此就会 crash 了。

一般情况下，我们平时要么就定义完就传出去了，要么就把 `block` 定义成了类的属性，所以就没有发生过这种情况。 

#### 堆块
平时对块的操作肯定不能以栈块的形式来存储啊。堆块，要在原来的基础上执行 `copy`，让代码保存在堆上。

```objc
void (^block)();

if ( /* some condition */ ) {
    block = [^{
         NSLog(@"Block A");
   } copy];
} else {
    block = [^{
         NSLog(@"Block B");
    } copy];
}

block();
```

然后 `block` 就能指向堆上的地址了。

平时我们用属性方式保存块的时候都是这样声明的：

```objc
@property (nonatomic,copy) Block block;
```

这个属性里暗含了 `copy` 操作了。

> block 会捕获外部的变量的值，然后将其复制为自己私有的 const 变量。所以一般不让在 block 内部改变外部变量的值。但是可以在外部变量前加上 `__block` 修饰，这样就会将外部变量的内存捕获，进而不管在 block 内部还是外部都可以修改变量的值。

#### 全局块
在全局内存里声明的就是全局块，没用过。不知道有什么好处。

### 第38条：为常用的块类型创建typedef
如果我们需要重复创建某种块（相同参数，返回值）的变量，我们就可以通过typedef来给某一种块定义属于它自己的新类型：

```objc
int (^variableName)(BOOL flag, int value) =^(BOOL flag, int value){
     // Implementation
     return someInt;
}

- (void)startWithCompletionHandler: (void(^)(NSData *data, NSError *error))completion;
```

这个块有一个 bool 参数和一个 int 参数，并返回 int 类型。我们可以给它定义类型：

```objc
typedef int(^EOCSomeBlock)(BOOL flag, int value);
typedef void(^EOCCompletionHandler)(NSData *data, NSError *error);

EOCSomeBlock block = ^(BOOL flag, int value){
     // Implementation
};

- (void)startWithCompletionHandler:(EOCCompletionHandler)completion;
```

### 第39条：用handler块降低代码分散程度
可以通过**块的方式代替代理模式**。

代理模式主要是为了让其他类在必要时候调用自己类的方法。而使用块的方式可以直接将方法内容作为参数或者属性传入调用块。这样设计业务逻辑更加直观清晰。

### 第40条：用块引用其所属对象时不要出现保留环
注意使用块的时候不要产生保留环，要在块执行完成后，将块置为 `nil`。

一种是传入 weak 对象，一种是执行完置 nil。我觉得还是传入 weak 对象比较好。因为我不知道这个块会不会执行。如果不执行，那不是一直释放不了了？

### 第41条：多用派发队列，少用同步锁
多个线程执行同一份代码时，很可能会造成数据不同步。作者建议使用 GCD 来为代码加锁的方式解决这个问题。

#### 方案一：使用串行同步队列来将读写操作都安排到同一个队列里：
```objc
_syncQueue = dispatch_queue_create("com.effectiveobjectivec.syncQueue", NULL);

//读取字符串
- (NSString*)someString {
    __block NSString *localSomeString;
    dispatch_sync(_syncQueue, ^{
        localSomeString = _someString;
    });
    return localSomeString;
}

//设置字符串
- (void)setSomeString:(NSString*)someString {
    dispatch_sync(_syncQueue, ^{
        _someString = someString;
    });
}
```

这里，用了一个串行队列，保证了读写操作都加了锁，是一种解决方式。但是，我们要明确一点，数据的正确性主要取决于写入操作，只要保证写入时，线程是安全的，那么即便读取操作是并发的，也可以保证数据是同步的。因此，我们要加以改进，读操作并行，写操作串行。可以通过  `dispatch_barrier_async`、`dispatch_barrier_sync` 完成。

#### 将写操作放入栅栏块中，让他们单独执行；将读取操作并发执行
在队列中，栅栏块必须单独执行，不能与其他块并行。这只对并发队列有意义，因为串行队列中的块总是按照顺序逐个执行。并发队列如果发现接下来要处理的块是个栅栏块，那么就一直等到当前所有并发块都执行完毕，才会单独执行这个栅栏块。待栅栏块执行过后，再按正常方式继续向下处理。**相当于给并行队列里加个锁**

```objc
_syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

//读取字符串
- (NSString*)someString {
    __block NSString *localSomeString;
    dispatch_sync(_syncQueue, ^{
       localSomeString = _someString;
    });
    return localSomeString;
}

//设置字符串
- (void)setSomeString:(NSString*)someString {

    dispatch_barrier_async(_syncQueue, ^{
        _someString = someString;
    });
}
```

这里解释下为什么读取用的是 `sync`，写入用的是 `async`，因为读取是要有返回值的，要将值返回给调用的对象，总不能还没有拿到值，就已经 `return` 了吧；而写入操作没有返回值，那么就让它立即 `return` 执行后面的代码，另开线程写入。

### 第42条：多用GCD，少用performSelector系列方法
在iOS开发中，有时会使用 `performSelector` 来执行某个方法，但是 `performSelector` 系列的方法能处理的选择子很局限，最好使用 GCD：
- 它无法处理带有多个参数的选择子（有最多支持两个选择子的方法）
- 返回值只能是void或者对象类型
- 会引起内存泄露

但是如果将方法放在块中，通过 GCD 来操作就能很好地解决这些问题。尤其是我们如果想要让一个任务在另一个线程上执行，最好应该将任务放到块里，交给 GCD 来实现，而不是通过 `performSelector` 方法。

#### 延后执行某个任务的方法
```objc
// 使用 performSelector:withObject:afterDelay:
[self performSelector:@selector(doSomething) withObject:nil afterDelay:5.0];


// 使用 dispatch_after
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5.0 * NSEC_PER_SEC));
dispatch_after(time, dispatch_get_main_queue(), ^(void){
    [self doSomething];
});
```

#### 将任务放在主线程执行
```objc
// 使用 performSelectorOnMainThread:withObject:waitUntilDone:
[self performSelectorOnMainThread:@selector(doSomething) withObject:nil waitUntilDone:NO];


// 使用 dispatch_async
// (or if waitUntilDone is YES, then dispatch_sync)
dispatch_async(dispatch_get_main_queue(), ^{
        [self doSomething];
});
```

如果 `waitUntilDone` 的参数是 `Yes`，那么就对应 GCD 的 `dispatch_sync` 方法。我们可以看到，使用 GCD 的方式可以将线程操作代码和方法调用代码写在同一处，一目了然；而且完全不受调用方法的选择子和方法参数个数的限制。

### 第43条：掌握GCD及操作队列的使用时机
除了 GCD，**操作队列（NSOperationQueue）**也是解决多线程任务管理问题的一个方案。对于不同的环境，我们要采取不同的策略来解决问题：有时候使用 GCD 好些，有时则是使用操作队列更加合理。（并不清楚操作队列怎么用的，反正就抄一下哪里好）

- 可以取消操作：在运行任务前，可以在NSOperation对象调用 `cancel` 方法，标明此任务不需要执行。但是 GCD 队列是无法取消的，因为它遵循“安排好之后就不管了（fire and forget）”的原则。
- 可以指定操作间的依赖关系：例如从服务器下载并处理文件的动作可以用操作来表示。而在处理其他文件之前必须先下载“清单文件”。而后续的下载工作，都要依赖于先下载的清单文件这一操作。
- 监控 `NSOperation` 对象的属性：可以通过 KVO 来监听 `NSOperation` 的属性：可以通过 `isCancelled` 属性来判断任务是否已取消；通过 `isFinished` 属性来判断任务是否已经完成。
- 可以指定操作的优先级：操作的优先级表示此操作与队列中其他操作之间的优先关系，我们可以指定它。

### 第44条：通过Dispath Group机制，根据系统资源状况来执行任务
有时需要等待多个并行任务结束的那一刻执行某个任务，这个时候就可以使用 `dispath group` 函数来实现这个需求：

通过 `dispath group` 函数，可以把并发执行的多个任务合为一组，于是调用者就可以知道这些任务何时才能全部执行完毕。

```objc
//一个优先级低的并发队列
dispatch_queue_t lowPriorityQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

//一个优先级高的并发队列
dispatch_queue_t highPriorityQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);

//创建dispatch_group
dispatch_group_t dispatchGroup = dispatch_group_create();

//将优先级低的队列放入dispatch_group
for (id object in lowPriorityObjects) {
 dispatch_group_async(dispatchGroup,lowPriorityQueue,^{ [object performTask]; });
}

//将优先级高的队列放入dispatch_group
for (id object in highPriorityObjects) {
 dispatch_group_async(dispatchGroup,highPriorityQueue,^{ [object performTask]; });
}

//dispatch_group里的任务都结束后调用块中的代码
dispatch_queue_t notifyQueue = dispatch_get_main_queue();
dispatch_group_notify(dispatchGroup,notifyQueue,^{
     // Continue processing after completing tasks
});
```

想要更详细的了解，还是看之前的 GCD 介绍文章吧。

### 第45条：使用dispatch_once来执行只需运行一次的线程安全代码
有时我们可能只需要将某段代码执行一次，这时可以通过 `dispatch_once` 函数来解决。

`dispatch_once` 函数比较重要的使用例子是单例模式：
我们在创建单例模式的实例时，可以使用 `dispatch_once` 函数来令初始化代码只执行一次，并且内部是线程安全的。

而且，对于执行一次的 `block` 来说，每次调用函数时传入的标记都必须完全相同，通常标记变量声明在 `static` 或 `global` 作用域里。

```objc
+ (id)sharedInstance {
     static EOCClass *sharedInstance = nil;
     static dispatch_once_t onceToken;
     dispatch_once(&onceToken, ^{
             sharedInstance = [[self alloc] init];
    });
     return sharedInstance;
}
```

### 第46条：不要使用dispatch_get_current_queue
已经被废弃的 API，不说了。

## 系统框架
### 第47条：熟悉系统框架
主要的系统框架：
- Foundation  :NSObject,NSArray,NSDictionary 等
- CFoundation :C 语言 API，Foundation 框架中的许多功能，都可以在这里找到对应的 C 语言 API
- CFNetwork   :C 语言 API，提供了 C 语言级别的网络通信能力 
- CoreAudio   :C 语言 API，操作设备上的音频硬件
- AVFoundation:提供的 OC 对象可以回放并录制音频和视频
- CoreData    :OC 的 API，将对象写入数据库
- CoreText    :C 语言 API，高效执行文字排版和渲染操作


### 第48条：多用块枚举，少用for循环
#### 传统的for遍历
```objc
NSArray *anArray = /* ... */;
for (int i = 0; i < anArray.count; i++) {
   id object = anArray[i];
   // Do something with 'object'
}



// Dictionary
NSDictionary *aDictionary = /* ... */;
NSArray *keys = [aDictionary allKeys];
for (int i = 0; i < keys.count; i++) {
   id key = keys[i];
   id value = aDictionary[key];
   // Do something with 'key' and 'value'
}


// Set
NSSet *aSet = /* ... */;
NSArray *objects = [aSet allObjects];
for (int i = 0; i < objects.count; i++) {
   id object = objects[i];
   // Do something with 'object'

}
```

我们可以看到，在遍历 NSDictionary,和 NSet 时，我们又新创建了一个数组。虽然遍历的目的达成了，但是却加大了系统的开销。

#### 利用快速遍历
```objc
NSArray *anArray = /* ... */;
for (id object in anArray) {
 // Do something with 'object'
}

// Dictionary
NSDictionary *aDictionary = /* ... */;
for (id key in aDictionary) {
 id value = aDictionary[key];
 // Do something with 'key' and 'value'

}


NSSet *aSet = /* ... */;
for (id object in aSet) {
 // Do something with 'object'
}
```

这种快速遍历的方法要比传统的遍历方法更加简洁易懂，但是缺点是无法方便获取元素的下标。

#### 利用基于块（block）的遍历
oc 提供了新的遍历方式，与 for 循环相比，能够优雅的获得元素的下标：

```objc
// array
- (void)enumerateObjectsUsingBlock:(void(^)(id object,NSUInteger idx,BOOL *stop))block
// set
- (void)enumerateObjectsUsingBlock:(void(^)(id object,BOOL *stop))block
// dic
- (void)enumerateObjectsUsingBlock:(void(^)(id key,id object,BOOL *stop))block
```

其中可以通过 `*stop = YES` 来中途终止遍历。注意一定要带 `*`。

后面几个没啥意思。用的不多，不写了。










































