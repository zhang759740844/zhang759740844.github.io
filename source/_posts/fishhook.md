title: fishhook 源码解析
date: 2019/7/7 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

提起 iOS 中的 hook 手段，我们最先想到的是 Method Swizzling，这是 hook OC 方法最重要的手段。fishhook 则提供了 hook 系统 c 函数的手段。

<!--more-->

## Lazy Binding 过程

### 为什么要有 Lazy Binding

fishhook 作用于系统符号的 Lazy Binding 过程中。Lazy Binding 的过程是将系统的动态库中的符号动态链接的过程。

为了减少应用的体积，加速应用的启动速度。苹果系统会将很多系统库设计为动态库。动态库的实际地址在应用编译的时候是未知的。一些符号会在应用启动的时候链接。但是如果这样的符号过多就会拖慢应用的启动速度。因此一些非必要符号会在第一次使用的时候绑定，也就是 Lazy Binding。

### 从例子看 Lazy Binding

我们来通过一个系统方法 NSLog 来验证动态库符号的绑定过程。我们在 main.m 中输入两个打印语句，然后加上断点：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook1.png?raw=true)

我们来看一下它的汇编执行过程，需要先设置：debug -> debug workflow -> always show disassembly。断点情况下的汇编代码如下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook2.png?raw=true)

我们看到第14行，`bl 0x10470ebf0； symbol stub for: NSLog` 就是执行打印的方法。这个方法对应于可执行文件的哪个段呢？我们通过 MachOView 查看它的可执行文件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook4.png?raw=true)

在 `__Text,__stubs` 中有一个对应于 NSLog 的方法。它的 offset 是 00006BF0。但是我们实际的地址是 0x10470ebf0，这两个地址显然是不匹配的。这是因为加载到内存中的可执行文件都会被加上一个随机数，即 ALSR 技术。这个地址每次加载都是随机的，因此如果你做了同样的尝试，肯定和我的不一样。我们可以通过 lldb 中执行 `image list` 查看这个地址：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook5.png?raw=true)

如上图所示，`image list` 能列举出当前应用加载的所有动态库。其中第一个就是当前应用的地址，也就是 0x0000000104708000

>  可执行文件 offset + ALSR 地址 = 实际内存中的地址
>
> 0x00006BF0 + 0x104708000 = 0x10470ebf0

那么这个地址上的方法是什么呢？可以在 lldb 中通过 `dis` 命令输出：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook3.png?raw=true) 

在 MachOView 中这个地址的数据为 `1F2003D5F0A0005800021FD6` 。这是一段16进制的机器码。我们可以把它转为 ARM 汇编代码验证一下是不是就是 `dis` 的输出。如果我们自己查表会非常费事，有一个 [ARM to Hex 的网站](http://armconverter.com/)，可以帮助我们将汇编代码转换为 Hex。我们只验证第二句汇编语句的正确性：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook7.png?raw=true)

通过对比可以发现，MachOView 中相应 offset 的 Data 反汇编后就是 `dis` 生成的汇编代码。

再来看一下反汇编后的汇编代码中调用的方法地址 0x10470ec2c 。我们把它转为 MachOView 中的 offset：0x6c2c。它的地址在 `__Text,__stub_helper` 中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook8.png?raw=true)

`__Text,__stub_helper` 会调用 `dyld_stu_binder` 方法计算并绑定 `NSLog` 函数的真实地址。

让程序走过这个方法，再次通过 `dis` 查看汇编代码：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook6.png?raw=true)

此次执行的方法地址不再是 0x10470ec2c，而是 0x2065176e6。说明 `__Text,__stub` 的方法指向发生了改变。

