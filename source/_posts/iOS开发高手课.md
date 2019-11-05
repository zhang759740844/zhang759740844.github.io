title: 《iOS开发高手课阅读笔记》
date: 2019/9/9 14:07:12  
categories: iOS
tags: 

 - 学习笔记
	
---

很早就买了戴铭老师的专栏，每一篇都是干货。也正是因为干货太多，所以不太愿意零碎的去学，怕知识点看过就忘了。因此，花了几个晚上，把感兴趣的高手课做了整理。

<!--more-->

## App 启动速度优化与监控

### 冷启动与热启动

- 冷启动是指， App 点击启动前，它的进程不在系统里，需要系统新创建一个进程分配给它启动的情况。这是一次完整的启动过程。 
- 热启动是指 ，App 在冷启动后用户将 App 退后台，在 App 的进程还在系统里的情况下，用户重新启动进入 App 的过程，这个过程做的事情非常少。

### App 启动三个阶段

1. main() 函数执行前；
2. main() 函数执行后；
3. 首屏渲染完成后。

#### main() 函数执行前

在 main() 函数执行前，系统主要会做下面几件事情：

1. 加载可执行文件（App 的.o 文件的集合）；
2. 加载动态链接库，进行 rebase 指针调整和 bind 符号绑定； 
3. Objc runtime 的初始处理，包括 Objc 相关类的注册、category 注册、selector 唯一性检查等；
4.  初始化，包括了执行 +load() 方法、attribute((constructor)) 修饰的函数的调用、创建 C++ 静态全局变量。

因此，我们可以通过相应的一下几种方式优化：

1. 减少动态库。尽量将动态库合并。
2. 减少不用的类或者方法。
3. +load() 方法里的内容可以放到首屏渲染完成后再执行，或使用 +initialize() 替换。
4. 控制 C++ 全局变量的数量

#### main() 函数执行后

main()函数执行后的阶段，指的是从main()函数执行开始，到appDelegate的didFinishLaunchingWithOptions方法里首屏渲染相关方法执行完成。

因此，我们应该考虑哪些初始化操作是首屏渲染必须的，哪些是对应功能开始时才需要初始化的。

#### 首屏渲染完成后

首屏渲染完成后，就要进行非首屏其他业务模块的初始化，监听的注册，配置文件的读取等。这个阶段就是从渲染完成开始，到 didFinishLaunchingWithOptions 方法作用域结束。

### 监控方法时长

- **定时抓取主线程上方法调用堆栈，计算一段时间里各个方法的耗时。** Xcode 中的 Time Profile 就是这么做的。时间间隔一般设置为 0.01s。间隔太长容易监听不到所有方法的执行。
- **对objc_msgSend 方法进行 hook 来掌握所有方法的执行耗时**。hook objc_msgSend 方法能够精确获取 OC 方法执行时间。对于 c 和 block，可以使用 libffi 的 ffi_call 完成 hook。

我们可以通过 fishhook hook objc_msgSend。fishhook 的原理可以看我之前的博客。

>  emmmm。一大段一大段的汇编代码有点难懂。。。暂时先知道通过这种方式可以实现方法调用时间就行了



## 链接器

### 为什么要有链接器？

没有这个绑定过程的话，单个文件生成的 Mach-O 文件是无法正常运行起来的。因为，如果运行时碰到调用在其他文件中实现的函数的情况时，就会找不到这个调用函数的地址，从而无法继续执行。

链接器在链接多个目标文件的过程中，会创建一个符号表，用于记录所有已定义的和所有未定义的符号。链接时如果出现相同符号的情况，就会出现“ld: dumplicate symbols”的错误信息；如果在其他目标文件里没有找到符号，就会提示“Undefined symbols”的错误信息。

### 链接器做了什么？

1. 去项目文件里查找目标代码文件里没有定义的变量。 
2. 扫描项目中的不同文件，将所有符号定义和引用地址收集起来，并放到全局符号表中。 
3. 计算合并后长度及位置，生成同类型的段进行合并，建立绑定。 
4. 对项目中不同文件里的变量进行地址重定位。

### 动态库链接

Mach-O 文件是编译后的产物，而动态库在运行时才会被链接，并没参与 Mach-O 文件的编译和链接，所以 Mach-O 文件中并没有包含动态库里的符号定义。也就是说，这些符号会显示为“未定义”，但它们的名字和对应的库的路径会被记录下来。运行时通过 dlopen 和 dlsym 导入动态库时，先根据记录的库路径找到对应的库，再通过记录的名字符号找到绑定的地址。 dlopen 会把共享库载入运行进程的地址空间，载入的共享库也会有未定义的符号，这样会触发更多的共享库被载入。

加载过程开始会修正地址偏移，iOS 会用 ASLR 来做地址偏移避免攻击，确定 Non-Lazy Pointer 地址进行符号地址绑定，加载所有类，最后执行 load 方法和 Clang Attribute 的 constructor 修饰函数。

### dyld 做了什么？

1. 先执行 Mach-O 文件，根据 Mach-O 文件里 undefined 的符号加载对应的动态库，系统会设置一个共享缓存来解决加载的递归依赖问题； 
2. 加载后，将 undefined 的符号绑定到动态库里对应的地址上； 
3. 最后再处理 +load 方法，main 函数返回后运行 static terminator。

## Crash

### 常见 Crash

信号可捕获崩溃：

- KVO
- NSNotification
- 数组越界
- 野指针
- ...

信号不可捕获崩溃

- 后台任务超时
- 内存溢出
- 主线程卡顿超阈值 (异常编码 0x8badf00d)
- ...

> 信号不可捕获的崩溃通常都是系统级的，系统杀掉 App 后会将日志保存在系统目录下。开发者没有权限获取系统目录日志。

### 如何收集捕获 不到的崩溃？

Background Task 可以通过 `beginBackgroundTaskWithExpirat:` 来延长后台执行时间。后台执行任务最多可以执行3分钟，超过3分钟就会被系统挂起。

```objc
- (void)applicationDidEnterBackground:(UIApplication *)application {
  self.backgroundTaskIdentifier = [application beginBackgroundTaskWithExpirationHandler:^(void) {
    [self yourTask];
  }];
}
```

对于此类异常可以先设置一个定时器，在接近3分钟时判断后台程序是否还在执行。如果还在执行，那么就可以判断程序即将崩溃，进行上报，记录，以达到监控的效果。

监控内存溢出和主线程卡顿被 watchdog 杀死，同样需要先找到一个阈值，然后在临近阈值时还在执行，判断为即将崩溃。内存溢出可以通过内存映射(mmap)来保存现场；主线程卡顿超过阈值，可以手机当前线程的堆栈信息。具体措施，看后续。

## 卡顿监控

### 利用 runloop 监控卡顿

卡顿的发生可以通过一次 runloop 从开始到结束的时间间隔来间接判断。

注册 observer 记录 runloop 开启的时间，并且在 runloop 结束的时候清空，然后创建一个子线程，每隔一定时间去检测当前时间和 runloop 开启时记录的时间是否大于某一个阈值。

当大于某个阈值的时候表明产生了卡顿，记录下卡顿时候的堆栈

### 利用 FPS 变化监控卡顿

这个实现方式类似 YYFPSLabel，通过创建一个 CADisplayLink，看看一秒钟回调能执行多少次。

> 这是我看 YYFPSLabel 源码学到的

### 子线程监控卡顿

首先创建一个子线程，然后在这个子线程中将下面的事件循环：

设置一个标志位，默认其表示主线程阻塞状态。然后通过 gcd 给主队列增加一个任务。这个任务用来清除这个标志位，将标志位置为非阻塞。也就是说，如果主线程没有堵塞，那么就会执行这个任务将是否阻塞的标志位置为非阻塞。

然后将子线程阻塞 threshold 秒，在线程醒来之后判断这个标志位是否置为了非阻塞状态。如果置为了非阻塞状态就说明在这 threshold 秒之内，主线程执行了任务，没有堵塞。如果标志位还是阻塞状态，说明主线程被阻塞了，没来得及执行我们的方法。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/caton_1.png?raw=true)

> 这是我看 Beehive 中关于 watchdog 部分学到的

## OOM 监控

### 获取内存上限

#### JetsamEvent 日志

查看 JetsamEvent 开头的日志(设置->隐私->分析) 通过日志中的 `pageSize` 和 `rpages` 字段的乘积得到内存上限。

#### XNU 获取

在 XNU 中，有专门用于获取内存上限值的函数和宏。可以通过 memorystatus_priority_entry 这个结构体，获进程优先级和内存限制：

```c
typedef struct memorystatus_priority_entry {
	pid_t pid;
	int32_t priority;
	uint64_t user_data;
	int32_t limit;
	uint32_t state;
} memorystatus_priority_entry_t;
```

priority 是优先级，limit 是内存限制

#### 内存警告获取

通过系统回调函数 `didReceiveMemoryWarning` 回调作为入口获取内存状况。以下代码通过 `task_info` 函数能够拿到当前内存占用：

```objc
struct mach_task_basic_info info;
mach_msg_type_number_t size = sizeof(info);
kern_return_t kl = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);

float used_mem = info.resident_size;
NSLog(@" 使用了 %f MB 内存 ", used_mem / 1024.0f / 1024.0f)
```

### 获取内存信息

通过 fishhook Hook `malloc_logger` 函数

## libffi

我们可以通过动态链接器提供的api `dlsym()` 传入函数名获得函数指针。

但是 `dlsym()` 获取的函数指针需要调用必须要提前声明好参数个数和类型。比如下方，2 就是错的。因为 1 在编译的时候编译器能知道有两个 int 类型的参数，就会在调用的时候将两个 int 参数入栈。而 2 没有声明，编译器就不会申请空间了。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jikeshijian_1.png?raw=true)

当然也可以把要调用的 c 函数都设置为 id 类型，然后生成 n 个不同参数个数的函数指针，通过 switch 判断参数个数的方式去赋值。JSPatch 也有通过这总方式实现的。

objc_msgSend 通过直接用汇编代码的方式完成函数体，这样就不需要通过编译器生成汇编代码了。通过约定好的方式去取参数是能够取得到的。

我们可以通过 libffi 动态调用任意 c 函数，也是通过汇编直接实现的.

## 内存泄漏监控

> 此为最近阅读相关源码的时候的总结。因为也是监控的一种，故放到这里来了。

这里列举两种内存泄漏监控的方式

### MLeaksFinder

微信读书内存泄漏的检测方法。

hook NavigationController 的 push 和 pop 方法，在 pop 的时候保存为 ViewController 中的每一个 View 以及自身添加定时器，2分钟后，执行内存泄漏方法，如果没有内存泄漏，那么这个对象为 nil，内存泄漏的方法无法执行。
这种方式是在销毁的生命周期的时候触发延时检查

### PLeakSniffer

这是 MrPeak 的检测方法

把 ViewController 设置为其实例对象的弱引用。如果实例对象没有销毁，但是 ViewController 已经置为 nil 了，就说明可能 leak 了。
只需要周期性的发送通知，各个实例对象接收到通知的时候执行方法，检查 ViewController 是否为 nil 即可知道是否是可能泄露的
这种方式是在创建的时候注册通知，定时轮询

[iOS内存泄漏自动检测工具PLeakSniffer](http://mrpeak.cn/blog/leak/)



