title: objective-c 学习笔记
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- Objective-C
	- 学习笔记

---

开始IOS开发必然要先学会objective-c，本篇是阅读了《object-c编程》一书后摘录的笔记，比较浅显。

<!--more-->

### 函数
函数可以有很多局部变量，这些变量都会保存在函数的帧中。这些帧存于栈内。执行函数时，函数的帧会在栈的顶部被创建。函数返回时，帧退出栈，等待下一个调用它的函数继续执行。

### 全局变量
在函数外声明的变量，只要类被import就能使用。  
Java中不存在这种全局变量，只能定义一个类，通过类名，资源名访问。  
为了防止混淆问题，引用静态变量。与全局变量一样，也需要在函数外声明。不过只有某个声明静态变量的文件才能访问。好处是：既保留了非局部的、存在于任何函数之外的优点，又避免了其他文件修改的问题。  
**那么这和@property有什么区别？**静态变量是持久的，存储于静态存储区，在程序刚运行时就被唯一一次初始化，不会像property随着实例化而变化。

### 指针
&表示取地址，*表示取值。  
空指针用nil而不是null。

### 结构
使用结构保存多个相关数据。
```objc
struct Person{
	float height；
	int age；
}；
```
使用：`struct Person mikey；`  
使用typedef简化。typedef可以为某类型声明一个等价的别名，该别名用法和常规数据类型无异。
```objc
typedef struct {
	float height；
	int age；
} Person；
```
使用：`Person mikey；`

### 获取结构中的属性：
当结构的使用者是一个指针时，使用->表示先获取指针p指向的数据结构，然后返回该结构的成员变量。  
当结构的使用者是一个实例时，使用.表示访问属性。

### id %@
id表示可以指向任意类型的指针。变量声明不使用星号。id已经隐含了星号的作用。
InstanceType表示方法的返回类型。  
%@表示占位符，代表指针，会向相应指针变量对象发送description消息。

### NSString
@“…”表示创建一个NSString对象。需要知道字符串完整内容。  
也可使用stringWithFormat方法动态创建：
```objc
NSString *dateString = [NSString stringWithFormat:@“The date is %@”, now]
```

### NSArray
创建：
```objc
NSArray *dateList = @[now, tomorrow ,yesterday];
```
NSArray是无法改变的，被创建后无法添加删除以及改变顺序。  
快速遍历数组： 
```objc
for(NSDate *d in daeList){}
```



### NSMutableArray
可变数组，可添加删除和修改顺序。  
insertObject：atIndex在指定位置插入  
removeObject：atIndex删除数组中对象  
快速遍历时不能添加删除数组内数据。

### 自定义一个类
头文件以@interface开始，@end结束。花括号内声明实例变量，实例变量以下划线”_”开始
取方法名字和相应实例变量一样，弹药去掉实例变量开头的下划线。存方法以set开头，后面更上去掉下划线的实例变量名。

### self
self是指针，指向运行当前方法的对象。

### 属性
属性的声明以@property开头，然后是类型和名称。属性自动声明存取方法。  
使用”.”获取属性其实是在发送消息，调用getset方法。语法糖。

### 继承
NSObject是所有类的基类，拥有一个实例变量：isa指针。任何一个对象的isa指针都指向创建爱你该对象的类。发送消息时，对象查询是否有该消息名的方法。没有则继续查询父类。父类也没有，查找参数超类。

### @class
一般来说，@class是放在interface中的，只是为了在interface中引用这个类，把这个类作为一个类型来用的。 在实现这个接口的实现类中，如果需要引用这个类的实体变量或者方法之类的，还是需要import在@class中声明的类进来.  
如果ClassA.h中仅需要声明一个ClassB的指针，那么就可以在ClassA.h中声明@ClassB

### 类拓展
类拓展是一组私有的声明。只有类和其实例才能使用在类拓展中声明的属性方法。
```objc
	#import “BNREmployee.h”
	@interface BNREmployee()	
	@property (nonatomic) unsigned int officeAlarmCode;
	@end
```
注意要有括号，写在implement前面。  
头文件中的属性相当于公有，可以通过同样公有的getset方法获取。类拓展无法则无法在其它类中直接获得，必须手动设置getset方法。

### 弱引用
通过弱引用可以解决强引用循环。强引用会保留对象的拥有方，使其不会释放。弱引用则不会保留。

### Collection类
NSSet对象包含的内容是无序的，并且在NSSet对象中，特定对象只能出现一次。其最大用处是检查某个对象是否存在。NSMutableSet可以添加删除set内数据。  
NSDictionary对象是键值对集合。字典的字面量语法由@和{}组成。字典里的键是独一无二的。  
NSMutable是NS的子类。  
collection不能保存nil。如果要保存nil则要保存NSNull类的实例。
```objc
[hotel addObject:[NSNull null]];
```

### 常量
可以通过两种途径定义常量，#define和全局变量。  
1. #define A B 告诉编译器看到A用B替换
2. extern NSString const *NSLocaleCurrencyCode；  
   const表示指针的不会变化,至于 `extern` 和 `static` 的用法，参见 “IOS 技巧”

### #include和#import
import会确保预处理器只导入特定的文件一次，include允许多次导入同一个文件。最好使用import。



### 协议
协议可以为一个对象指定角色。类似于接口。如果某个对象要扮演特定的角色，就一定要实现相应的必须方法，并选择实现部分可选方法。  
UITableView数据源协议是UITableViewDataSource，方法声明如下：
```objc
@protocal UITableViewDataSource<NSObject>
@required
- (NSInteger)tableView:(UITableView *)tv
	numberOfRowsInSection:(NSInteger) section;
@optional
……. 
```

### 范畴
通过范畴（category）可以为任何已有的类添加方法。  
创建一个新文件，类型为Objective-c category，将新范畴命名为BNRVowelCounting，对应类为NSString。  
打开NSString+BNRVowelCounting.h为范畴声明一个方法。该方法会被加入NSString类。
声明示例：NSString+MD5.h
```objc
@interface NSString (MD5)
+ (NSString *)md5:(NSString *)originalStr;
@end
```


