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

那么这个地址上的方法是什么呢？可以在 lldb 中通过 `dis` 命令反汇编输出：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook3.png?raw=true) 



在 MachOView 中这个地址的数据为 `1F2003D5F0A0005800021FD6` 。这是一段16进制的机器码。我们可以验证一下它和反汇编的结果是否一致。如果我们自己查表会非常费事，有一个 [ARM to Hex 的网站](http://armconverter.com/)，可以帮助我们将汇编代码转换为 Hex。我们只验证第二句汇编语句的正确性：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook7.png?raw=true)

通过对比可以发现，MachOView 中相应 offset 的 Data 反汇编后就是 `dis` 生成的汇编代码。

重新看一下反汇编后的第二三条汇编代码。他们的作用是加载相对于当前地址，偏移量为 `0x141c` 的内存到寄存器 x16 中，然后执行。实际执行的地址的计算过程如下：

> 0x141c + 0x6BF4 = 0x8010
>
> 第一条汇编代码 offset 为 0x6BF0，那么第二条汇编代码 offset 即为 0x6BF4

在 MachOView 中体现为 `__DATA,__la_symbol_ptr`：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook9.png?raw=true)

在 0x8010 的地址上的 data 为 0x0000000100006C2C，静态分析的地址，默认会有 0x100000000 的偏移，这在 TEXT 段的 Load Commands 中有所体现：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook10.png?raw=true)

所以实际的偏移地址为 0x6C2C。它的地址在 `__Text,__stub_helper` 中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook8.png?raw=true)

`__Text,__stub_helper` 会调用 `dyld_stu_binder` 方法计算并绑定 `NSLog` 函数的真实地址。

让程序走过这个方法，再次通过 `dis` 查看汇编代码：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook6.png?raw=true)

此次执行的方法地址不再是 0x10470ec2c，而是 0x2065176e6。也就是说进过了绑定之后， `__Text,__stub` 的方法指向发生了改变，指向了系统的动态库的方法。至此完成了 lazy binding 的过程。

## fishhook

通过上面的 lazy binding 的例子分析，我们能知道，懒绑定的符号在 `__DATA，_la_symbol_ptr` 段上，会在第一次调用的时候绑定实际的地址。而一般 c 函数的地址在编译的时候就已经确定了，位于程序的 `TEXT` 段，为只读区域。

这种懒绑定的方式其实叫做 PIC(地址无关代码)，fishhook 能够影响这部分符号地址的绑定。

### 使用

fishhook 的使用需要创建一个 `rebinding` 结构体，结构体中需要包含要 hook 的函数的名称，要替换的方法实现，被 hook 的方法的容器：

```objc

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
 
    // 定义rebinding 结构体
    struct rebinding rebind = {};
  	// 需要hook的函数名称
    rebind.name = "NSLog";
  	// 新函数的地址
    rebind.replacement = hookNSLog;
  	// 原始函数地址的指针
    rebind.replaced = (void *)&sys_NSLog;
    //将上面的结构体 放入 reb结构体数组中
    struct rebinding rebindObj[]  = {rebind};
    
    /*
     * arg1 : 结构体数据组
     * arg2 : 数组的长度
     */
    rebind_symbols(rebindObj, 1);
}

//定义一个函数指针 用于指向原来的NSLog函数
static void (*sys_NSLog)(NSString *format, ...);

void hookNSLog(NSString *format, ...){
    format = [format stringByAppendingString:@"被勾住了"];
    sys_NSLog(format);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    funcDlog(@"原有NSLog函数");
}

@end

```

### 源码解析

先从调用方法 `rebind_symbols` 方法入手：

```c
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
    // 维护一个 rebindings_entry 的结构
    // 将 rebinding 的多个实例组织成一个链表
    int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
    // 判断是否 malloc 失败，失败会返回 -1
    if (retval < 0) {
        return retval;
    }
    // _rebindings_head -> next 是第一次调用的标志符，NULL 则代表第一次调用
    if (!_rebindings_head->next) {
        // 第一次调用，将 _rebind_symbols_for_image 注册为回调
        _dyld_register_func_for_add_image(_rebind_symbols_for_image);
    } else {
        // 先获取 dyld 镜像数量
        uint32_t c = _dyld_image_count();
        for (uint32_t i = 0; i < c; i++) {
            // 根据下标依次进行重绑定过程
            _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
        }
    }
    // 返回状态值
    return retval;
}
```





