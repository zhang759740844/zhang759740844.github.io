title: Effective Objective-C 学习笔记
date: 2016/9/8 10:07:12  
categories: iOS
tags:
	- Objective-C

---

拜读一下 Effective Objective-C 这本书，做一些笔记

<!--more-->

## 熟悉OC
### 第1条：了解Objective-C的起源

对于消息结构的语言，运行时所执行的代码由运行环境来决定；在运行时才回去查找索要执行的方法。其实现原理是由**运行期组件（runtime component）**完成，使用 Objective-C 的面向对象特性所需的全部数据结构以及函数都在运行期组件里面。

运行期组件本质上是一种与开发者所编写的代码相链接的动态库（dynamic library），其代码能把开发者所编写的所有程序粘合起来。

### 第2条： 在类的头文件中尽量少引用其他头文件

有时，类A需要将类B的实例变量作为它公共 API 的属性。这个时候，我们不应该引入类B的头文件，而应该使用**向前声明（forward declaring）** 使用 `@class` 关键字，并且**在 A 的实现文件引用 B 的头文件**。

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
2. 可以避免循环引用：因为如果两个类在自己的头文件中都引入了对方的头文件，那么就会导致其中一个类无法被正确编译。

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

注意这里的 `const`, 如果在 `*` 前面，表示指针指向的地址是不能改变的，如果在 `*` 后面，表示指针指向的堆上的内容不能改变。 

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

在枚举类型的 switch 语句中不要实现 default 分支。它的好处是，当我们给枚举增加成员时，编译器就会提示开发者：switch 语句并未处理所有的枚举。否则添加了枚举却没有实现 switch 将可能导致严重的崩溃。

## 对象、消息、运行期
### 第6条：理解“属性”这一概念
#### 存取方法
在设置完属性后，编译器会自动写出一套存取方法，用于访问相应名称的变量。访问属性，可以使用点语法。编译器会把点语法转换为对存取方法的调用。如果我们不希望编译器自动生成存取方法的话，需要设置 `@dynamic` 字段：

```objc
@interface EOCPerson : NSManagedObject

@property NSString *firstName;
@property NSString *lastName;

@end


@implementation EOCPerson
@dynamic firstName, lastName;
@end
```

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

### 第7条： 在对象内部尽量直接访问实例变量














































