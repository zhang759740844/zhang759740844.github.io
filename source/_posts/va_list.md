title: iOS 中 va_list 的使用
date: 2017/12/22 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

看 FMDB 源码的时候看到了 `va_list` 的使用。这个东西的主要用途就是不确定个数的入参。一般我们的入参不确定的时候，我们会使用一个 dic 来保存数据，当然简单的方式就是使用本篇要说的这个。FMDB 提供了不确定参数个数的 sql 语句的调用方法。这个功能其实很类似于系统的 `NSLog`。

<!--more-->

### 使用方式

先来看一下 FMDB 的使用场景(略有修改)：

```objc
- (BOOL)executeUpdate:(NSString*)sql, ... {
    va_list args;
    va_start(args, sql);
    ...
    
    [self getStringWithVAList:(va_list)args];
    
   	...
    va_end(args);
    return result;
}

- (void)getStringWithVAList:(va_list)args {
  	while (index < queryCount) {
		queryString = va_arg(args, NSString*);
      	index+=1;
	}
}
```

### 解释

简而言之，`va_list` 的原理就是：函数的参数是存储在栈中的，因此我们可以通过不停的移动指针，来获取栈中的所有元素。

上面是 FMDB 更新 sql 的执行方法。这个方法先接受一个 sql 字符串，之后是可变的参数。可变参数会在方法中填入 sql 里。

`va_list args` 其实相当于声明了一个类型为 `va_list` 名为 `args` 的变量。

`va_start(args, sql)` 表示的是取到 `sql` 后一个参数的地址交给 `args`。通过这一步后，你就可以将把 `args` 当做一个变量任意传递了。

`va_arg(args, NSString*)` 可以获得当前指向的参数，并且将指针移动到后一个参数。指针并不知道要移动多少个字节，所以第二个参数传入一个类型，交代指针的偏移量。所以这里要注意的是，不同的类型偏移量是不同的，所以你**必须事先知道每一个可变参数的类型，然后为不同类型设置不同偏移**。比如说 FMDB，甚至 `NSLog`，都需要在第一个参数中为不同类型设置不同的占位符，如 `%d`,`%@`。在处理的时候就会从前往后扫描字符串，解析这些占位符，以此获得可变参数的类型。还要注意的是，**我们需要自己精确控制循环次数**，无限制的 `va_arg()` 的调用会产生越界。所以综上，**`va_list` 不会知道我们参数的数据类型和个数，这就是为什么我们需要占位符的原因**。

`va_end(args)` 表示清空 `args`。