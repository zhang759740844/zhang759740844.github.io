title: Block 的使用
date: 2016/9/1 10:07:12  
categories: iOS
tags:
 - Objective-C

----

类似于匿名函数，oc中提供`block`,可以将一段代码块像对象一样作为参数传递、执行。

<!--more-->

## Block的使用
### Block实例
```objc
^(double dividend){
	double quotient = dividend / divisor;
	return quotient;
}
```
Block对象可以被当做一个实参来传递给可以接收block的方法。

### 声明block变量:
```objc 
void (^devowelizer)(id, NSUInteger, BOOL*);
```
`void` 表示返回类型  
`^` 表示是一个block对象  
`devowelizer` 表示block变量的名称  
后面的是实参类型  
方法的调用参数类型为`^(id  string, NSUInteger i, BOOL *stop)block`

### 编写Block对象
```objc
devowelizer = ^(id string,NSUInteger i, BOOL *stop){
	……
};
```

### 调用block变量
```objc
devowelizer(string,i,stop);
```
### typedef
不能在方法的实现代码中使用typedef，需要在实现文件的顶部，或者头文件内使用typedef。
```objc
typedef void(^ArrayEnumerationBlock)(id,NSUInteger,BOOL *);
```
需要注意的是，这里定义的是一个新的类型，不是变量。跟在^后面的是类型名称。创建这个新类型后，可以简化相应Block的声明。
```objc
ArrayEnumerationBlock devowelizer；
```

### 外部变量
在执行Block对象时，为了确保其下的外部变量能够始终存在，相应的Block对象会捕获这些变量，意味着程序会拷贝变量的值。

**当外部对象是引用时，block会复制其引用的地址(指针指向的堆上的地址)，只能改变地址指向的对象的属性，不能将外部对象指向新的地址.**

### 修改外部变量
如果需要在Block对象内修改某个外部变量，则可以声明相应的外部变量时，在前面加上__block关键字。

### 在Block中使用self
如果要写一个使用self的Block对象，需要避免强引用循环。  
在Block外声明一个_weak指正，然后将这个指针指向Block对象使用的self，最后在Block对象中使用这个新的指针。
```objc
_weak typeof(self) weakSelf = self;	//弱引用指针
	myBlock = ^{
		NSLog(@“Employee:%@”,weakSelf);
	};
```

## block的实现

### block 结构

**block本质上也是一个oc对象，他内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。**

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_1.png?raw=true)

将上图和下图做对照，可以发现，两者的结构基本一致：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_13.png?raw=true)

> 注意看上面的 block 结构体，在结构体内部也创建了一个同名的变量 age，它的值和外部的 age 相同。因此，内部 age 修改了并不会影响外部 age。

### block 变量捕获

变量有五种形式，auto 局部变量、static 局部变量、auto 全局变量、static 全局变量、成员变量。

#### 为什么要有变量的捕获

因为如果是引用外部局部变量，局部变量会在作用域结束后回收。局部变量回收了，闭包肯定就 crash 了。因此需要进行变量的捕获。

#### 变量如何捕获的

捕获方式是在内部创建一个同名的变量，赋给它相同的值或者指向相同的地址。这样虽然代码上看起来是同一个变量，但是其实是两个变量。

#### 为什么不让使用者直接修改使用到的外部变量

上面说到，block 中使用的外部变量其实不是正真的外部变量，而是新创建的和外部同名的变量。因此，就算可以改，改的也只是内部的同名变量的指向。因此，编译器索性就不让改了。

#### 局部变量

对于局部变量来说，auto 局部变量捕获的是值，static 局部变量捕获的是地址。这是因为 auto 局部变量可能会销毁，所以要捕获值，否则会访问已回收的地址。而 static 局部变量则不用担心变量回收，所以捕获的是地址。***所以，在block调用之前修改地址中保存的值，block中的 static 变量会随之改变***。例：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_2.png?raw=true)

#### 全局变量

block不需要捕获全局变量，因为全局变量无论在哪里都可以访问。**局部变量因为跨函数访问所以需要捕获，全局变量在哪里都可以访问 ，所以不用捕获。**

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_3.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_4.png?raw=true)

#### 成员变量

成员变量是比较特殊的，**即使block中使用的是实例对象的属性，block中捕获的仍然是实例对象，并通过实例对象 get 方法去获取使用到的属性**

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_6.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_5.png?raw=true)

> 因为点语法实质是调用方法，所以捕获的是 self

### block 的类型

#### 三种类型

block 存在三种类型：

```objc
__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）
__NSStackBlock__ （ _NSConcreteStackBlock ）
__NSMallocBlock__ （ _NSConcreteMallocBlock ）
```

我们可以通过代码查看他们的类型：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_7.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_8.png?raw=true)

#### 三种类型的定义

**没有访问auto变量的block是*`__NSGlobalBlock__`*类型的，存放在数据段中。访问了auto变量的block是`__NSStackBlock__`类型的，存放在栈中。`__NSStackBlock__`类型的block调用copy成为*`__NSMallocBlock__`类型并被复制存放在堆中。**

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_9.png?raw=true)

栈 block 相当于一个局部变量，当超出作用域，就面临着被回收的风险。ARC 下，很多情况中，会自动帮我们执行一次 copy，比如赋值。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_10.png?raw=true)

> 因此，block 的属性修饰词需要使用 copy。当然 ARC 环境下，使用 strong，编译器也会自动帮我们 copy。

### __weak 关键字

`__weak` 关键字修饰的变量会对引用对象进行弱引用。在 block 中使用被弱引用的变量，block 内部也会捕获弱引用的对象：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_11.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_12.png?raw=true)

### __block 关键字

上面说到，block 内部会创建一个和外部同名的变量具有相同的值，或者指向相同的地址。那么这样就无法做到外部修改外部变量的值，block 内部做相应的变化；block 内部修改变量的值，外部变量做相应的变化。

那么如何才能做到内外的修改能够同步呢？见下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_14.png?raw=true)

赋值操作都会引起外部变量和内部变量的表示的值或地址的不同。要解决这种不同只有通过一层中间层来解决。**外部变量和内部变量都指向相同的地址。而对于外部变量或者内部变量的修改都转为对包裹的对象的修改，而不是切换外部变量或者内部变量的指向。**

iOS 提供了  `__block` 来达到这个目的。`__block` 修饰的变量，编译器会自动将其包装为一个对象，原本被修饰的变量会作为对象的一个属性。

> 只有要修改变量的值或者地址的时候才使用` __block`，比如修改 NSMutableArray 这种则不需要使用 `__block`

### block 的嵌套

对于 block 的嵌套，比如以下场景，blockB会捕获self，这是大家都了解的，但是blockA会捕获self么？为什么？

```objc
- (void)embeddedBlock
{
    void (^blockA)() = ^{
        void (^blockB)() = ^{
            NSLog(@"%@",self);
        };
    };
}
```

答案是会，先将self从外界传入到blockA中。再从blockA的中传入到blockB中。blockB只能从blockA的作用域里捕获变量。因此blockB中捕获的任何东西，blockA必须也捕获一份。

## 参考

[iOS底层原理总结 - 探寻block的本质（一）](https://www.jianshu.com/p/c99f4974ddb5)

[iOS底层原理总结 - 探寻block的本质（二）](https://www.jianshu.com/p/8865ff43f30e)