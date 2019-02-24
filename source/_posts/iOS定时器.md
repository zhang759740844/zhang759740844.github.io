title: iOS 的各种定时器
date: 2017/2/3 11:07:12
categories: iOS
tags:

	- 学习笔记
---

这篇将探究一下 iOS 上的四种定时器的实现

<!--more-->

## NSTimer

一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop 为了节省资源，并不会在非常准确的时间点回调这个 Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

### 创建

有两种创建方式，方法一：

```objc
self.timer = [NSTimer timerWithTimeInterval:5 target:self selector:@selector(showTime) userInfo:nil repeats:YES];
// 必须加
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
// 如果是非主线程要调用,主线程已经自动运行 runloop 就不需要自己 run 了
[[NSRunLoop currentRunLoop] run];
```

这个类方法创建出来的对象如果不用 `addTimer: forMode` 方法手动加入循环池中，将不会循环执行。

这里如果不是用的 `NSRunLoopCommonModes` 而是 `NSDefaultRunLoopMode` 那么当界面滑动时，无法执行 `showTime` 方法回调。当滑动停止时，立刻执行最初的注册的定时事件，之后由于滑动导致未能注册的事件的回调一律忽略。

方法二：

```objc
self.timer = [NSTimer scheduledTimerWithTimeInterval:3.0 target:self selector:@selector(run:) userInfo:@"abc" repeats:NO];

// 下面这两句用来修改模式，如果不想修改默认模式，可以不加
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
// 如果是非主线程要调用,主线程已经自动运行 runloop 就不需要自己 run 了
[[NSRunLoop currentRunLoop] run];
```

这个方法会将定时器添加到当前的运行循环，运行循环的模式为默认模式。虽然指定了默认模式，但是还是允许你修改模式的。

和 `CADisplayLink` 一样，`NSTimer` 也需要手动取消定时器：

```objc
//取消定时器  
[self.timer invalidate]; 
self.timer = nil;    
```

> NSRunLoop 会强引用 NSTimer，所以外部不需要强引用 timer。

### 执行N次后结束

可以在外部创建一个计数器，当执行到一定次数后在内部结束：

```swift
var runCount = 0

Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { timer in
    print("Timer fired!")
    runCount += 1
    if runCount == 3 {
        timer.invalidate()
    }
}
```

### 附加 context

创建 timer 时传入的 userInfo 是一个 id 类型。所以可以直接通过 timer 拿到 userInfo 代表的值，需要强转类型：

```swift
let context = ["user": "@twostraws"]
Timer.scheduledTimer(timeInterval: 1.0, target: self, selector: #selector(fireTimer), userInfo: context, repeats: true)


@objc func fireTimer(timer: Timer) {
    guard let context = timer.userInfo as? [String: String] else { return }
    let user = context["user", default: "Anonymous"]

    print("Timer fired by \(user)!")
    runCount += 1

    if runCount == 3 {
        timer.invalidate()
    }
}
```

### NSTimer 不准时

NSTimer 是由 RunLoop 控制执行的，当到达时间后，会触发睡眠的 RunLoop 执行定时器方法。但是如果当前 RunLoop 一直在运行，则无法立刻执行定时器方法，导致延时执行。

## CADisplayLink

### 基本用法

`CADisplayLink` 是一个能让我们**以和屏幕刷新率同步的频率将特定的内容画到屏幕上的定时器类**。` CADisplayLink` 以特定模式注册到 runloop 后， 每当屏幕显示内容刷新结束的时候， runloop 就会向 `CADisplayLink` 指定的 target 发送一次指定的 selector 消息，  `CADisplayLink` 类对应的 selector 就会被调用一次。 

使用方式：

```objc
- (void)startDisplayLink
{
    self.displayLink = [CADisplayLink displayLinkWithTarget:self
                                                   selector:@selector(handleDisplayLink:)];
    self.displayLink.frameInterval = 3;
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop]
                           forMode:NSDefaultRunLoopMode];
}

- (void)handleDisplayLink:(CADisplayLink *)displayLink
{
    //do something
}

- (void)stopDisplayLink
{
  	// 注意，下面两步都要做。先从 runloop 里注销 displaylink。再将 displaylink 置为 nil,防止你对已经 invalidate 的对象进行操作。
    [self.displayLink invalidate];
    self.displayLink = nil;
}
```

当把 `CADisplayLink` 对象 add 到 runloop 中后（一定要加啊，否则无效），selector就能被周期性调用，类似于 `NSTimer` 被启动了；执行 invalidate 操作时， `CADisplayLink` 对象就会从 runloop 中移除，selector 调用也随即停止，类似于 `NSTimer` 的 invalidate 方法。

iOS 设备的 FPS 是60Hz，因此 `CADisplayLink` 的 selector 默认调用周期是每秒60次，这个周期可以通过 `frameInterval` 属性设置， `CADisplayLink`的 selector 每秒调用次数为 `60/ frameInterval`。比如当 `frameInterval` 设为2，每秒调用就变成30次。因此， `CADisplayLink` 周期的设置方式略显不便。不过 `CADisplayLink` 适合于需要精度较高的定时。

> NSRunLoop 会强引用 CADisplayLink，所以外部也不需要强引用 CADisplayLink 

### 应用之查看 FPS

CADisplayLink 和屏幕的刷新率是一致的，所以可以通过判断一秒钟内 CADisplayLink 执行了多少次来间接知道屏幕刷新率：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cadisplaylink.png?raw=true)

### 与 NSTimer 不同

**iOS**设备的屏幕刷新频率是固定的，`CADisplayLink`在正常情况下会在每次刷新结束都被调用，精确度相当高。
`NSTimer`的精确度就显得低了点，比如`NSTimer`的触发时间到的时候，`runloop`如果在阻塞状态，触发时间就会推迟到下一个`runloop`周期。并且 `NSTimer`新增了`tolerance`属性，让用户可以设置可以容忍的触发的时间的延迟范围。
`CADisplayLink`使用场合相对专一，适合做UI的不停重绘，比如自定义动画引擎或者视频播放的渲染。`NSTimer`的使用范围要广泛的多，各种需要单次或者循环定时处理的任务都可以使用。

> 可以通过 `CADisplayLink` 和上面的绘图，完成很多动画效果



## 延迟调用

当调用 NSObject 的 `performSelector:afterDelay:` 后，**实际上其内部会创建一个 Timer 并添加到当前线程的 Run Loop 中**。**所以如果当前线程没有 Run Loop，则这个方法会失效**（注意，这是一个考点，所以在子线程中直接调用该方法是无效的）。

当调用 `performSelector:onThread:` 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 Run Loop 该方法也会失效。

所以一般还是在主线程调用这些方法，如果实在要在子线程里调用，那么记得在其后调用 `[[NSRunLoop currentRunLoop] run];`

```objc
// 3秒后自动调用self的run:方法，并且传递参数：@"abc"
[self performSelector:@selector(run:) withObject:@"abc" afterDelay:3];
// 在子线程中创建的时候，要开启子线程的 runloop
[[NSRunLoop currentRunLoop] run];
```

> 本质上就是一个 NSTimer

## GCD

```objc
/** 创建定时器对象
 * para1: DISPATCH_SOURCE_TYPE_TIMER 为定时器类型
 * para2-3: 中间两个参数对定时器无用
 * para4: 最后为在什么调度队列中使用
 */
dispatch_source_t _gcdTimer; //(不是指针)
_gcdTimer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_global_queue(0, 0));
/** 设置定时器
 * para2: 任务开始时间
 * para3: 任务的间隔
 * para4: 可接受的误差时间，设置0即不允许出现误差
 * Tips: 单位均为纳秒
 */
dispatch_source_set_timer(_gcdTimer, DISPATCH_TIME_NOW, 2.0 * NSEC_PER_SEC, 0.0 * NSEC_PER_SEC);
/** 设置定时器任务
 * 可以通过block方式
 * 也可以通过C函数方式
 */
dispatch_source_set_event_handler(_gcdTimer, ^{
    static int gcdIdx = 0;
    NSLog(@"GCD Method: %d", gcdIdx++);
    NSLog(@"%@", [NSThread currentThread]);
    
    if(gcdIdx == 5) {
        // 终止定时器
        dispatch_suspend(_gcdTimer);
    }
});
// 启动任务，GCD计时器创建后需要手动启动
dispatch_resume(_gcdTimer);
```

GCD 是准时的定时器

> 注意 `dispatch_source_t` 并不是一个指针

> `dispatch_source_t` 本身需要强引用，否则会立刻回收，无法完成定时

##  关于 runloop

我们可以观察到，除了 GCD，另外三种方法都需要把定时器加入当前 runloop 中。这里要强调一点，一般情况下，不要把定时器加到非主线程中去。因为定时器需要线程的 runloop 启动，即 `[[NSRunLoop currentRunLoop] run];`。对于主线程，这没有任何问题，主线程的 runloop 默认开启。但是**对于子线程，我们最好不要启动它的 runloop，这会导致子线程一直处于活动状态而不被回收，从而造成内存泄漏**。所以，定时器最好还是加在主线程里吧。**如果一定要在子线程里设置，那么一定要在结束的时候，停止运行循环**。



## 关于循环引用问题

定时器包括 CADisplayLink 和  NSTimer 都需要手动通过 `invalidate` 方法来销毁。这就关系到销毁的时机。

### 方式一

通过 `dealloc` 方法销毁：

错误的做法：

```objc
__weak typeof(self) wself = self;
self.displayLink = [CADisplayLink displayLinkWithTarget:wself selector:@selector(selector)];
[self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];

- (void)dealloc {
    [self.displayLink invalidate];
}
```

上面这种弱引用方式是没有用的。因为它们内部会强引用 target。所以其实不会执行 dealloc 方法。

为了打破这种强引用，我们可以新建一个 weakTarget 的类，通过转发实现：

```objc
// class WeakTarget
@property (nonatomic, weak) id target;

- (instancetype)initWithTarget:(id)target {
    self = [super init];
    if (self) {
        self.target = target;
    }
    return self;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    return self.target;
}
```

使用：

```objc
self.displayLink = [CADisplayLink displayLinkWithTarget:[[WeakTarget alloc] initWithTarget:self] selector:@selector(selector)];
[self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];

- (void)dealloc {
    [self.displayLink invalidate];
}
```

这样，timer 强引用的就不是我们业务相关的类了。

### 方式二

通过 VC 或者 View 的生命周期销毁。虽然强引用了不会调用 `dealloc` ，但是生命周期还是会回调的比如在 `viewWillDisappear` 或者 `removeFromSuperView` 等方法中。