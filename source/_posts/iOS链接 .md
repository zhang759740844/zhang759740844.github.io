title: Xcode other link flags 分析
date: 2017/6/19 10:07:12  
categories: iOS
tags:
	- Xcode
---

导入第三方库的时候，经常会看到要设置 “other link flags”。那么这到底是有什么用？

<!--more-->

### 基础
#### 编译连接
从C代码到可执行文件经历的步骤是：
```
源代码 > 预处理器 > 编译器 > 汇编器 > 机器码 > 链接器 > 可执行文件
```

经过编译汇编之后输出的文件是`.o`文件，然后对这些`.o`文件进行链接，就是最终的可执行文件。静态链接库一般是一个`.a` 压缩文件，解压后可以得到众多的`.o`文件，静态库也会在链接过程中连接到你的可执行文件中去。



#### 符号

我们在源码中写的全局变量名、函数名、类名在生成的 `.o` 文件中都可以叫做符号，存在一个符号表中。



#### OC 链接特性

> The “selector not recognized” runtime exception occurs due to an issue between the implementation of standard UNIX static libraries, the linker and the dynamic nature of Objective-C. Objective-C does not define linker symbols for each function (or method, in Objective-C) - instead, linker symbols are only generated for each class. If you extend a pre-existing class with categories, the linker does not know to associate the object code of the core class implementation and the category implementation. This prevents objects created in the resulting application from responding to a selector that is defined in the category.

Object-C的链接器并**不会为每个方法建立符号表，而是为每个类建立链接符号**。这样的话静态库中定义了已存在的类的分类，链接器就以为这个类存在了，不会将分类和核心类代码关联（合并）起来，这样在最后可执行文件中，就会找不到分类里所定义的方法。





### 三个参数

#### -ObjC

> This flag causes the linker to load every object file in the library that defines an Objective-C class or category. While this option will typically result in a larger executable (due to additional object code loaded into the application), it will allow the successful creation of effective Objective-C static libraries that contain categories on existing classes.

加入这个参数后，链接器会将静态库中的每个类和分类按需加载到最后的可执行文件，当然，这个参数会导致可执行文件比较大，原因是加载了更多的额外对象的代码到可执行文件当中去，但是这会解决 Objec-C 中静态库中已存在的类包含的分类问题。

> Important: For 64-bit and iPhone OS applications, there is a linker bug that prevents -ObjC from loading objects files from static libraries that contain only categories and no classes. The workaround is to use the -allload or -forceload flags.

这里还提到，链接器有一个bug，就是当静态库中只有 category 没有 class 的时候，必须使用 -all_load 或者 -force_load 代替。



#### -all_load:

该参数把所找到的目标文件都加载到可执行文件当中去（-ObjC 是按需加载），但是这就存在一个问题了，如果两个静态库中，都使用了同一份可执行文件（这是一个很常见的问题，例如大家的目标文件都使用了用以名字 base64.o）就会发生 ld: duplicate symbol 符号冲突问题，所以不太建议使用。



#### -force_load：

该参数的作用跟 -all_load其实是一样的，但是 -force_load 需要指定要进行全部加载的库文件的路径，这样的话，只要完全加载一个库文件，不影响其余库文件的按需加载。



### 原理

在对静态库进行链接的时候，链接器会获取静态库的符号表。只有被使用了的类才能才会被链接到可执行文件中去。比如你有50个类文件，但是只有20个是使用到的，那么只有这20个会被链接，其它30个被忽略。

这种方式对于C或者C++是非常有效的，因为这些静态语言会在编译的时候就已经把该做的工作都做完了。但是 OC 是一门动态语言，很多 OC 的特性要在运行时决定。因此，链接器不能得知其是否是被使用到的。category 就是一个运行时特性，category 不是像 class 那样的符号，所以链接器不会直接把 category 链接到可执行文件中去。通过 **-ObjC** 可以告知链接器，在按需加载的前提下，将 category 也链接进可执行文件。







[Xcode other link flags 详解](http://simplecodesky.com/2017/06/14/Xcode-other-link-flags-详解/)

[[Objective-C categories in static library](https://stackoverflow.com/questions/2567498/objective-c-categories-in-static-library)] 中 Vladimir 和 Mecki 的回答