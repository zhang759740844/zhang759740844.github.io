title: runLoop学习笔记
date: 2016/11/1 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

runLoop 虽然平时用不到，但是面试的时候问的多啊。那我也就来了解一下 runLoop 的原理。

<!--more-->

## 概念
### 什么是RunLoop
以下是一个 iOS 程序的 main 函数：

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

main 函数是程序的入口，那么为什么程序执行完毕后没有退出呢？因为 RunLoop，使线程循环，能够随时处理事件但并不退出。这种方式在各个框架中都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

上面代码中 `UIApplicationMain()` 方法在这里不仅完成了初始化我们的程序并设置程序 Delegate 的任务，而且随之开启了主线程的 RunLoop，开始接受处理事件。这样我们的应用就可以在无人操作的时候休息，需要让它干活的时候又能立马响应。流程图如下所示：

![runloop_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_1.png?raw=true)

在 OS X/iOS 系统中，提供了两个这样的对象：
- **CFRunLoopRef**：是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
- **NSRunLoop**：是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

先来一张 RunLoop 机制关系图总览：

![runloop_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_2.png?raw=true)

### RunLoop与线程的关系
苹果不允许直接创建 RunLoop，提供了两个获取函数，CFRunLoopRef 的获取方法为 `CFRunLoopGetMain()`， `CFRunLoopGetCurrent()`。NSRunLoop 的获取方法是 `currentRunLoop`,`mainRunLoop`。NSRunLoop 对象可以通过 `getCFRunLoop` 方法获得 CFRunLoopRef 对象。CFRunLoopRef 的内部逻辑如下：

```objc
/// 全局的 Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 Run Loop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 Run Loop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 Run Loop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```

从上面的代码可以看出，线程和 RunLoop 之间是一一对应的(这也就解释了前面关系图中 CFRunLoop 和 Thread 连线中的两个`1`的意义)，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有，**所以一个子线程，你想要它有 RunLoop 就必须在该线程内调用 `NSRunLoop *runLoop =[NSRunLoop currentRunLoop]`。如果你想启动这个 RunLoop，则要继续调用 `[runLoop run]**`。**但是注意，一般不需要开启子线程的 runLoop，因为这会让子线程一直存在，不会回收。**RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。**你只能在一个线程的内部获取其 RunLoop（主线程除外）**。

推荐一个链接[深入研究 runloop 与线程保活](https://bestswifter.com/runloop-and-thread/)



## RunLoop对外接口
在 CoreFoundation 里面关于 RunLoop 有5个类:
- CFRunLoopRef
- CFRunLoopModeRef
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef

其中，CFRunLoopModeRef 类并没有对外暴露，不能直接得到其对象，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:
![runloop_3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_3.png?raw=true)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

### 接口类型
#### RunLoop的Source
RunLoop 对象处理的事件源分为两种：Input sources 和 Timer sources（分别对应上面的 CFRunLoopSourceRef 和 CFRunLoopTimerRef，统称为事件源）：
- Input sources：用分发异步事件，通常是用于其他线程或程序的消息，比如：`performSelector:onThread:`
- Timer sources：用分发同步事件，通常这些事件发生在特定时间或者重复的时间间隔上，比如：`[NSTimer scheduledTimerWithTimeInterval:target:selector:]`

下图是苹果官方的一张图，展示了 RunLoop 的概念结构及各种事件源
![runloop_4](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_4.png?raw=true)

##### Input Source
Input Source 也就是 CFRunLoopSourceRef 有两个版本:Source0(Custom Input Sources)和 Source1(Port-Based Sources)：
- Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 `CFRunLoopSourceSignal(source)`，将这个 Source 标记为待处理，然后手动调用 `CFRunLoopWakeUp(runloop)` 来唤醒 RunLoop，让其处理这个事件。
- Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。

##### Time Source
基于时间的触发器，它和 NSTimer 是 Toll-Free Bridging 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop 会注册对应的时间点，当时间点到时，RunLoop 会被唤醒以执行那个回调。

Foundation 中 NSTimer Class 提供了相关方法来设置 Timer sources。需要注意的是除了 `scheduledTimerWithTimeInterval` 开头的方法创建的 Timer 都需要手动添加到当前 RunLoop 中。（`scheduledTimerWithTimeInterval` 创建的 Timer 会自动以 Default Mode 加载到当前 RunLoop中。）

#### RunLoop的Mode
RunLoop Mode 是指要被监听的事件源（包括 Input sources 和 Timer sources）的集合 + 要被通知的 run-loop observers 的集合。每一次运行自己的 RunLoop 时，都需要显示或者隐示的指定其运行于哪一种 Mode。在设置 RunLoop Mode 后，你的 RunLoop 会自动过滤和其他 Mode 相关的事件源，而只监视和当前设置 Mode 相关的源（通知相关的观察者）。大多数时候，RunLoop 都是运行在系统定义的默认模式上。

系统默认定了一下几个 mode：
- kCFRunLoopDefaultMode：App：App的默认 Mode，通常主线程是在这个 Mode 下运行的
- UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
- UIInitializationRunLoopMode：在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用（私有）
- GSEventReceiveRunLoopMode：接受系统事件的内部 Mode，通常用不到
- kCFRunLoopCommonModes：这是一个占位的 Mode，包含上面的 kCFRunLoopDefaultMode 以及 UITrackingRunLoopMode

应用场景举例：主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为"Common"属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入 CommonModes 中。

Mode 暴露的管理 mode item 的接口有下面几个：

```objc
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```

#### RunLoop的Observers
对比上面说的事件源，它们是在特定的同步事件或异步事件发生时被触发，RunLoop Observers 就不一样了，它是在 RunLoop 执行自己的代码到某一个指定位置时被触发。我们可以用 RunLoop Observers 来跟踪到这些事件：
- 进入 RunLoop 的时候
- RunLoop 将要处理一个 Timer source 的时候
- RunLoop 将要处理一个 Input source 的时候
- RunLoop 将要休眠的时候
- RunLoop 被唤醒，并准备处理唤醒它的事件的时候
- RunLoop 将要退出的时候

### 使用方式
#### Input Source 
#### Timer Source
#### Observers

这几个方面看起来太麻烦了，而且真的用不到，所以不想看了。=。=55555。贴几个介绍如何使用的博客吧，如果真的要用到了再看。
[RunLoop深度探究（五）](http://yangchao0033.github.io/blog/2016/01/18/runloop-5/)
[RunLoop](http://blog.imemo8.com/2016/04/17/RunLoop/)
[走进Run Loop的世界 (一)：什么是Run Loop？](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/)
[iOS多线程编程指南（三）Run Loop](https://www.dreamingwish.com/article/ios-multithread-program-runloop-the.html)
相关使用 Demo 也放到 github 上了。

## 实现
### 处理过程
![runloop_5](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_5.png?raw=true)

```objc
/// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}
 
/// 用指定的Mode启动，允许设置RunLoop超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
 
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}
```

可以看到，实际上 RunLoop 就是这样一个函数，其内部是一个 do-while 循环。当你调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里；直到超时或被手动停止，该函数才会返回。

### RunLoop 的底层实现
从上面代码可以看到，RunLoop 的核心是基于 mach port 的，其进入休眠时调用的函数是 `mach_msg()`。为了实现消息的发送和接收，`mach_msg()` 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数 `mach_msg_trap()`，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 `mach_msg_trap()` 时会触发陷阱机制，切换到内核态；内核态中内核实现的 `mach_msg()` 函数会完成实际的工作，如下图：
![runloop_6](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_6.png?raw=true)

**RunLoop 的核心就是一个 `mach_msg()`，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。例如你在模拟器里跑起一个 iOS 的 App，然后在 App 静止时点击暂停，你会看到主线程调用栈是停留在 `mach_msg_trap()` 这个地方。**

关于具体的如何利用 mach port 发送信息，哈哈，这个就不是我研究的东西了。

## 应用
苹果使用 RunLoop 实现了诸多功能

### AutoreleasePool
Autorelease 对象什么时候释放？答案当然不是“当前作用域大括号结束时释放”。在没有手加 Autorelease Pool 的情况下，Autorelease 对象是在当前的 runloop 迭代结束时释放的，而它能够释放的原因是系统在每个 runloop 迭代中都加入了自动释放池 Push 和 Pop。

#### 一个例子

```objc
__weak id reference = nil;
- (void)viewDidLoad {
    [super viewDidLoad];
    NSString *str = [NSString stringWithFormat:@"sunnyxx"];
    // str是一个autorelease对象，设置一个weak的引用来观察它
    reference = str;
}
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"%@", reference); // Console: sunnyxx
}
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"%@", reference); // Console: (null)
}
```

可以看到，在 `viewDidLoad` 方法后，一直到 `viewWillAppear` 字符串 reference 都没有被销毁，直到 `viewDidAppear` 方法后，才置为 `null`(这里创建字符串用的是 `stringWithFormat`，是在堆中创建一个对象，而不是在常量区)。由于这个 vc 在 `loadView` 之后便 add 到了 window 层级上，所以 `viewDidLoad` 和 `viewWillAppear` 是在同一个 `runloop` 调用的，因此在 `viewWillAppear` 中，这个 autorelease 的变量依然有值。

当然，我们也可以手动干预Autorelease对象的释放时机：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"sunnyxx"];
    }
    NSLog(@"%@", str); // Console: (null)
}
```

#### AutoreleasePool 原理
ARC下，我们使用 `@autoreleasepool{}` 来使用一个 AutoreleasePool，随后编译器将其改写成下面的样子：

```objc
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```

而这两个函数都是对 `AutoreleasePoolPage` 的简单封装，所以自动释放机制的核心就在于这个类。

`AutoreleasePoolPage` 是一个 C++ 实现的类
![runloop_7](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_7.png?raw=true)

- AutoreleasePool 并没有单独的结构，而是由若干个 AutoreleasePoolPage 以双向链表的形式组合而成（分别对应结构中的 parent 指针和 child 指针）
- AutoreleasePool 是按线程一一对应的（结构中的 thread 指针指向当前线程）
- AutoreleasePoolPage 每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存 autorelease 对象的地址
- 上面的 `id *next` 指针作为游标指向栈顶最新 add 进来的 autorelease 对象的下一个位置
- 一个 AutoreleasePoolPage 的空间被占满时，会新建一个 AutoreleasePoolPage 对象，连接链表，后来的 autorelease 对象在新的 page 加入

所以，若当前线程中只有一个 AutoreleasePoolPage 对象，并记录了很多 autorelease 对象地址时内存如下图：
![runloop_8](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_8.png?raw=true)

图中的情况，这一页再加入一个 autorelease 对象就要满了（也就是 next 指针马上指向栈顶），这时就要执行上面说的操作，建立下一页 page 对象，与这一页链表连接完成后，新page的next指针被初始化在栈底（begin的位置），然后继续向栈顶添加新对象。所以，向一个对象发送 `autorelease` 消息，就是将这个对象加入到当前 AutoreleasePoolPage 的栈顶 next 指针指向的位置。

每当进行一次 `objc_autoreleasePoolPush` 调用时，runtime 向当前的 AutoreleasePoolPage 中 add 进一个哨兵对象，值为0（也就是个 nil），那么这一个 page 就变成了下面的样子：
![runloop_9](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_9.png?raw=true)

`objc_autoreleasePoolPush` 的返回值正是这个哨兵对象的地址，被 `objc_autoreleasePoolPop` (哨兵对象)作为入参，于是：
1. 根据传入的哨兵对象地址找到哨兵对象所处的 page
2. 在当前 page 中，将晚于哨兵对象插入的所有 autorelease 对象都发送一次 `- release` 消息，并向回移动 next 指针到正确位置
3. 补充2：从最新加入的对象一直向前清理，可以向前跨越若干个 page，直到哨兵所在的 page

刚才的 `objc_autoreleasePoolPop` 执行后，最终变成了下面的样子：
![runloop_10](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_10.png?raw=true)


### 事件响应
苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 `__IOHIDEventSystemClientQueueCallback()`。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。 SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的 App 进程。随后苹果注册的那个 Source1 就会触发回调，并调用 `_UIApplicationHandleEventQueue()` 进行应用内部的分发。

`_UIApplicationHandleEventQueue()` 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

### 手势识别
当上面的 `_UIApplicationHandleEventQueue()` 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop 即将进入休眠) 事件，这个 Observer 的回调函数是 `_UIGestureRecognizerUpdateObserver()`，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行 GestureRecognizer 的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 界面更新
当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的  `setNeedsLayout/setNeedsDisplay` 方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()` 。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

这个函数内部的调用栈大概是这样的：

```objc
ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];
```

### 定时器

参见 iOS 定时器 这一篇。几乎所有定时器都要求 runloop 开启，且将自身加入到 runloop 中。

### 关于网络请求

iOS 中，关于网络请求的接口自下至上有如下几层:
- CFSocket
- CFNetwork       ->ASIHttpRequest
- NSURLConnection ->AFNetworking
- NSURLSession    ->AFNetworking2, Alamofire

![runloop_11](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runloop_11.png?raw=true)

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。


### AFNetworking
AFURLConnectionOperation 这个类是基于 NSURLConnection 构建的，其希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：

```objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 `[runLoop run]` 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 `NSMachPort (mach_port)` 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。

```objc
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```

当需要这个后台线程执行任务时，AFNetworking 通过调用 `[NSObject performSelector:onThread:..]` 将这个任务扔到了后台线程的 RunLoop 中。


## 参考
[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
[初识 Run Loop](http://itangqi.me/2016/04/14/the-first-meet-with-runloop/?utm_source=tuicool&utm_medium=referral)
[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)

