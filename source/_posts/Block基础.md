title: Block 的使用
date: 2016/9/1 10:07:12  
categories: iOS
tags:
	- Objective-C
---

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
block的实现主要参考[唐巧谈Objective-C block的实现](http://blog.devtang.com/2013/07/28/a-look-inside-blocks/#)

### block的结构
block的结构如下:
```objc
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};
```

结构图如下：
![block结构图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/block_struct.jpg?raw=true)

通过该图，我们可以知道，一个 block 实例实际上由 6 部分构成：
1. `isa` 指针，所有对象都有该指针，用于实现对象相关的功能。
2. `flags`，用于按 bit 位表示一些 block 的附加信息。
3. `reserved`，保留变量。
4. `invoke`，函数指针，指向具体的 block 实现的函数调用地址。
5. `descriptor`， 表示该 block 的附加描述信息，主要是 size 大小，以及 copy 和 dispose 函数的指针。
6. 各种从外部复制过来的变量，block 能够访问它外部的局部变量，就是因为将这些变量（或变量的地址）复制到了结构体中。

