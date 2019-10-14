title: YYAsyncLayer 源码解析
date: 2019/2/15 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

这篇是继 YYCache 后的 YYKit 源码解析。YYAsyncLayer 又是一个短小却质量奇高的轮子。非常适合学习和借鉴。

<!--more-->

## 使用

YYAsyncLayer 可以提供一个绘制自定义 CALayer 的自定义 UIView 组件。使用方式在其 [Github](<https://github.com/ibireme/YYAsyncLayer>)上有完整的体现：

```objc
@interface YYLabel : UIView
@property NSString *text;
@property UIFont *font;
@end

@implementation YYLabel

- (void)setText:(NSString *)text {
    _text = text.copy;
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)setFont:(UIFont *)font {
    _font = font;
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)contentsNeedUpdated {
    // do update
    [self.layer setNeedsDisplay];
}

#pragma mark - YYAsyncLayer

+ (Class)layerClass {
    return YYAsyncLayer.class;
}

- (YYAsyncLayerDisplayTask *)newAsyncDisplayTask {
    
    // capture current state to display task
    NSString *text = _text;
    UIFont *font = _font;
    
    YYAsyncLayerDisplayTask *task = [YYAsyncLayerDisplayTask new];
    task.willDisplay = ^(CALayer *layer) {
        //...
    };
    
    task.display = ^(CGContextRef context, CGSize size, BOOL(^isCancelled)(void)) {
        if (isCancelled()) return;
        NSArray *lines = CreateCTLines(text, font, size.width);
        if (isCancelled()) return;
        
        for (int i = 0; i < lines.count; i++) {
            CTLineRef line = line[i];
            CGContextSetTextPosition(context, 0, i * font.pointSize * 1.5);
            CTLineDraw(line, context);
            if (isCancelled()) return;
        }
    };
    
    task.didDisplay = ^(CALayer *layer, BOOL finished) {
        if (finished) {
            // finished
        } else {
            // cancelled
        }
    };
    
    return task;
}
@end
```

总的来说需要以下几点：

- 自定义的 UIView 实现 `YYAsyncLayerDelegate` 协议。
- 重写自定义 UIView 会引起视图更新的方法，在其中手动调用 `YYTransaction` 的方法，手动设置 layer `setNeedsDisplay`。
- 重写 UIView 的 layerClass 方法，返回 `YYAsyncLayer` 类型。
- 实现 `YYAsyncLayerDelegate` 协议 require 的方法 `newAsyncDisplayTask`，在其中创建一个 `YYAsyncLayerDisplayTask` 实例。提供三个 block：`willDisplay`，`display`，`didDisplay`。在 `display` 中通过 Core Graphic 绘图。



## 如何实现界面流畅

### 为何产生卡顿

计算机系统中 CPU、GPU、显示器是以上面这种方式协同工作的。CPU 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。

在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如**视图的创建、布局计算、图片解码、文本绘制**等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行**变换、合成、渲染**。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/asynclayer1.png?raw=true)

因此，要解决卡顿问题，就要从 CPU 和 GPU 两个角度完成。

### CPU 资源消耗原因

#### 对象的创建与销毁

对象的创建于销毁会消耗 CPU 资源。我们可以：

- 推迟创建，以及异步创建和销毁对象
- 复用对象
- 使用 CALayer 代替 UIView

关于对象的异步销毁，ibireme 提供了一种方式，把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁：

```objc
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
    [tmp class];
});
```

#### 布局计算与 AutoLayout

频繁的手动布局计算也会占用 CPU 的资源。因此，比如 tableview，就鼓励缓存 cell 的高度。

AutoLayout 在 iOS12 前效率比较低，不过在 iOS 12后已经进行了算法的优化，和直接设置 frame 差距不大。

#### 文本渲染

常用控件的文本渲染都是通过 CoreText 在主线程绘制 **Bitmap** 进行的。因此， 可以通过自定义控件，使用 TextKit 或者 CoreText 异步渲染

#### 图片解码

图片在提交到 GPU 之前才会被解码，并且是主线程中。因此，可以后台线程把图片绘制到 CGBitmapContext 中，然后通过 Bitmap 创建图片。

#### 图像的绘制

由于 CoreGraphic 方法通常都是线程安全的，所以图像的绘制可以很容易的放到后台线程进行。

```objc
- (void)display {
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

> 有一篇 《内存恶鬼drawRect》 的文章，讲述了 drawRect 会在重复创建 Bitmap 并且没有及时释放的时候会消耗大量的内存。
>
> 不过这是针对大范围的绘图的情况，通过 CAShapLayer 矢量图的方式而不是 Bitmap 有助于缓解内存压力

### GPU 资源消耗原因

#### 视图混合

多个视图重叠在一起的时候需要 GPU 把他们混合在一起。可以适当减少视图数量以及层级。并且避免使用存在透明度的视图。同时可以预先将多个视图渲染为一个图片显示。

#### 图像的合成

避免离屏渲染

## 实现

### 结构

YYAsyncLayer 的结构非常简单。只有三个类文件：

- `YYAsyncLayer`
- `YYSentinel`
- `YYTransaction`

`YYAsyncLayer` 继承于 CALayer，在内部重写了 `display` 方法，用来异步绘制。

`YYSentinel` 是一个自增的计数类。每次即将渲染的时候都会递增，并记录下来。当开始异步渲染的时候判断是否变化，来决定是否取消上次的渲染。

`YYTransaction` 给主线程 runloop 添加 observer 回调。用于在 runloop 空闲的时候进行渲染。

### YYTransaction

#### 创建

我们来看它的创建方法：

```objc
/// 创建 YYTransaction 实例，需要传入 target 和 selector
+ (YYTransaction *)transactionWithTarget:(id)target selector:(SEL)selector{
    if (!target || !selector) return nil;
    YYTransaction *t = [YYTransaction new];
    t.target = target;
    t.selector = selector;
    return t;
}
```

创建方法很普通，在生成的 `YYTransaction` 实例中存入 `target` 和 `selector`，也就是之后要执行的对象和方法。

#### 提交

提交方法中添加了主线程 runloop 的 `kCFRunLoopBeforeWaiting | kCFRunLoopExit` 时刻的监听：

```objc
- (void)commit {
    if (!_target || !_selector) return;
    ///
    YYTransactionSetup();
    /// 将 YYTransaction 实例保存到 Set 中
    [transactionSet addObject:self];
}

/// 在主线程 waiting 或者 exit 的时候，执行 transactionSet 中的方法
static void YYRunLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (transactionSet.count == 0) return;
    NSSet *currentSet = transactionSet;
    transactionSet = [NSMutableSet new];
    [currentSet enumerateObjectsUsingBlock:^(YYTransaction *transaction, BOOL *stop) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [transaction.target performSelector:transaction.selector];
#pragma clang diagnostic pop
    }];
}

/// 添加 observer 监听主线程 runloop 进入 waiting 和 exit
static void YYTransactionSetup() {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        transactionSet = [NSMutableSet new];
        CFRunLoopRef runloop = CFRunLoopGetMain();
        CFRunLoopObserverRef observer;
        
        observer = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                           kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                           true,      // repeat
                                           0xFFFFFF,  // after CATransaction(2000000)
                                           YYRunLoopObserverCallBack, NULL);
        CFRunLoopAddObserver(runloop, observer, kCFRunLoopCommonModes);
        CFRelease(observer);
    });
}
```

创建一个 Set，保存所有要执行的方法，然后将任务添加到主线程的 runloop 空闲时候执行。这是一个非常好的优化思想。非常值得学习。

### YYSentinel

这是一个自增的计数器。通过 `OSAtomicIncrement32` 方法实现递增：

```objc
@implementation YYSentinel {
    int32_t _value;
}

- (int32_t)value {
    return _value;
}

/// 自动累加
- (int32_t)increase {
    return OSAtomicIncrement32(&_value);
}

@end
```

`OSAtomicIncrement32()`是原子自增方法，线程安全。在日常开发中，若需要保证整形数值变量的线程安全，可以使用 OSAtomic 框架下的方法，它往往性能比使用各种“锁”更为优越，并且代码优雅。

### YYAsyncLayer

#### 初始化

`YYAsyncLayer` 继承于 CALayer。前面在设置 UIView 子类的时候需要设置 `layerClass` 方法返回 `YYAsyncLayer.class` 就是为了在创建 UIView 的时候，使用 `YYAsyncLayer` 作为其 CALayer。创建的时候会调用 `YYAsyncLayer` 的初始化方法：

```objc
- (instancetype)init {
    self = [super init];
    static CGFloat scale; //global
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        /// 物理像素 / 逻辑像素
        scale = [UIScreen mainScreen].scale;
    });
    /// UIView 和 UIImageView 默认处理了内部 CALayer 的 contentsScale。如果是直接使用 CALayer 及其衍生类，就需要显示的配置 contentScale
    self.contentsScale = scale;
    _sentinel = [YYSentinel new];
    /// 异步加载的标识
    _displaysAsynchronously = YES;
    return self;
}
```

它会根据屏幕的像素比设置 `contentsScale`。

#### 渲染方法

UIView 是持有 CALayer，并且作为 CALayer 的 delegate 的存在。当 UIView 即将渲染的时候，会调用 CALayer 的 `display` 方法。代码比较长，不过已经做了详尽的注释：

```objc
/// CALayer 的显示方法
- (void)display {
    super.contents = super.contents;
    [self _displayAsync:_displaysAsynchronously];
}

#pragma mark - Private

- (void)_displayAsync:(BOOL)async {
    /// CALayer 的 delegate 一般是 UIView 及其子类
    /// 自己创建的 UIView 需要实现 YYAsyncLayerDelegate 协议
    __strong id<YYAsyncLayerDelegate> delegate = (id)self.delegate;
    /// 获取绘制任务管理类。这个管理类由 CALayer 的 delegate 提供，需要设置 willDIsplay，display，didDisplay 这几个回调
    YYAsyncLayerDisplayTask *task = [delegate newAsyncDisplayTask];
    /// 如果没有 display 方法，那么把 contents 设置为 nil
    if (!task.display) {
        if (task.willDisplay) task.willDisplay(self);
        self.contents = nil;
        if (task.didDisplay) task.didDisplay(self, YES);
        return;
    }
    
    /// 是否是异步渲染
    if (async) {
        if (task.willDisplay) task.willDisplay(self);
        YYSentinel *sentinel = _sentinel;
        int32_t value = sentinel.value;
        BOOL (^isCancelled)(void) = ^BOOL() {
            return value != sentinel.value;
        };
        CGSize size = self.bounds.size;
        BOOL opaque = self.opaque;
        CGFloat scale = self.contentsScale;
        CGColorRef backgroundColor = (opaque && self.backgroundColor) ? CGColorRetain(self.backgroundColor) : NULL;
        /// 如果宽高等于0，那么释放 CALayer 的 contents
        if (size.width < 1 || size.height < 1) {
            CGImageRef image = (__bridge_retained CGImageRef)(self.contents);
            self.contents = nil;
            if (image) {
                dispatch_async(YYAsyncLayerGetReleaseQueue(), ^{
                    CFRelease(image);
                });
            }
            if (task.didDisplay) task.didDisplay(self, YES);
            CGColorRelease(backgroundColor);
            return;
        }
        
        /// 异步执行绘制，根据当前的自增值，获取串行队列
        dispatch_async(YYAsyncLayerGetDisplayQueue(), ^{
            /// 异步执行的时候，保存一个自增的值。如果异步执行的时候发现自增的值变化了，那么就说明之前的渲染已经被取消了
            /// 这个时候释放 backgroundColor 返回
            if (isCancelled()) {
                CGColorRelease(backgroundColor);
                return;
            }
            /// 绘制
            UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
            CGContextRef context = UIGraphicsGetCurrentContext();
            // 处理有透明度的情况
            if (opaque) {
                CGContextSaveGState(context); {
                    if (!backgroundColor || CGColorGetAlpha(backgroundColor) < 1) {
                        CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
                        CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
                        CGContextFillPath(context);
                    }
                    if (backgroundColor) {
                        CGContextSetFillColorWithColor(context, backgroundColor);
                        CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
                        CGContextFillPath(context);
                    }
                } CGContextRestoreGState(context);
                CGColorRelease(backgroundColor);
            }
            // 执行 UIView 中提供的 block
            task.display(context, size, isCancelled);
            /// 如果渲染完成后发现取消了。那么直接 return
            if (isCancelled()) {
                UIGraphicsEndImageContext();
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            /// 渲染完成，拿出渲染的视图
            UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            /// 如果获取完 image 后返现取消了，那么直接 return
            if (isCancelled()) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            /// 到主线程中设置 contents
            dispatch_async(dispatch_get_main_queue(), ^{
                /// 发现到主线程中的时候取消了，那么直接 return
                if (isCancelled()) {
                    if (task.didDisplay) task.didDisplay(self, NO);
                } else {
                    /// 没有取消设置 CALayer 的 contents
                    self.contents = (__bridge id)(image.CGImage);
                    if (task.didDisplay) task.didDisplay(self, YES);
                }
            });
        });
    } else {
        /// 同步执行直接自增
        [_sentinel increase];
        /// 之后就是正常的渲染逻辑
        if (task.willDisplay) task.willDisplay(self);
        UIGraphicsBeginImageContextWithOptions(self.bounds.size, self.opaque, self.contentsScale);
        CGContextRef context = UIGraphicsGetCurrentContext();
        if (self.opaque) {
            CGSize size = self.bounds.size;
            size.width *= self.contentsScale;
            size.height *= self.contentsScale;
            CGContextSaveGState(context); {
                if (!self.backgroundColor || CGColorGetAlpha(self.backgroundColor) < 1) {
                    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
                    CGContextAddRect(context, CGRectMake(0, 0, size.width, size.height));
                    CGContextFillPath(context);
                }
                if (self.backgroundColor) {
                    CGContextSetFillColorWithColor(context, self.backgroundColor);
                    CGContextAddRect(context, CGRectMake(0, 0, size.width, size.height));
                    CGContextFillPath(context);
                }
            } CGContextRestoreGState(context);
        }
        task.display(context, self.bounds.size, ^{return NO;});
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        self.contents = (__bridge id)(image.CGImage);
        if (task.didDisplay) task.didDisplay(self, YES);
    }
}
```

主要做了以下几点：

- 取出自定义 UIView 中实现的 `newAsyncDisplayTask` 方法返回的实例。从中取出 display 以及前后的回调。
- 判断要渲染的宽或者高为 0，那么直接清空 contents
- 如果是异步，那么创建异步队列，并通过 CoreGraphic 渲染。
- 如果是同步，直接同步渲染。

可以看到这里作者使用了大量的 `if (isCancelled()) {...}` 判断，只要失败，立刻返回。这样能够尽可能多的节省 CPU 资源。

来看一下 `isCancelled` 这个 block 的实现方式：

```objc
BOOL (^isCancelled)(void) = ^BOOL() {
    return value != sentinel.value;
};
```

通过 block 捕获当前渲染周期内的变量值。其他线程改变了全局变量`_sentinel`的值也不会影响当前的`value`。若当前`value`不等于最新的`_sentinel .value`时，说明绘制任务已经更新，当前绘制任务已经被放弃，就需要及时的做返回逻辑。

那么什么时候这个计数会改变呢？在提交重绘的时候。

```objc
- (void)setNeedsDisplay {
    [self _cancelAsyncDisplay];
    [super setNeedsDisplay];
}
- (void)_cancelAsyncDisplay {
    [_sentinel increase];
}
```

#### 异步线程的管理

每次异步渲染获取的队列并不是一个并行队列。而是多个串行队列：

```objc
static dispatch_queue_t YYAsyncLayerGetDisplayQueue() {
/// 如果使用了 YYDispatchQueuePool，用 YYDispatchQueuePool
#ifdef YYDispatchQueuePool_h
    return YYDispatchQueueGetForQOS(NSQualityOfServiceUserInitiated);
#else
#define MAX_QUEUE_COUNT 16
    static int queueCount;
    static dispatch_queue_t queues[MAX_QUEUE_COUNT];
    static dispatch_once_t onceToken;
    static int32_t counter = 0;
    dispatch_once(&onceToken, ^{
        /// 串行队列数量和处理器数量相同
        queueCount = (int)[NSProcessInfo processInfo].activeProcessorCount;
        queueCount = queueCount < 1 ? 1 : queueCount > MAX_QUEUE_COUNT ? MAX_QUEUE_COUNT : queueCount;
        /// 循环创建串行队列，bing并设置优先级
        if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0) {
            for (NSUInteger i = 0; i < queueCount; i++) {
                dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, 0);
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", attr);
            }
        } else {
            for (NSUInteger i = 0; i < queueCount; i++) {
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", DISPATCH_QUEUE_SERIAL);
                dispatch_set_target_queue(queues[i], dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
            }
        }
    });
    /// 根据自增的值返回某个串行队列
    int32_t cur = OSAtomicIncrement32(&counter);
    if (cur < 0) cur = -cur;
    return queues[(cur) % queueCount];
#undef MAX_QUEUE_COUNT
#endif
}
```

为什么不用并行队列呢？因为并行和并发是有区别的。在单核设备上，CPU通过频繁的切换上下文来运行不同的线程，速度足够快以至于我们看起来它是‘并行’处理的，然而我们只能说这种情况是并发而非并行。

并行队列并不能完全体现出多核处理器的优势。实际上一个 n 核设备同一时刻最多能 **并行** 执行 n 个任务，也就是最多有 n 个线程是相互不竞争 CPU 资源的。

而串行队列中只有一个线程，该框架中，作者使用和处理器核心相同数量的串行队列来轮询处理异步任务，有效的减少了线程调度操作。

### ASDK

ASDK 的渲染部分也是通过异步的方式进行渲染。不过 ASDK 还在其他方面进行了优化，包括，Layout，Rendering 和 UIKit Objects。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/asynclayer_2.png?raw=true)

## 总结

YYAsyncLayer 的代码也就几百行，非常的精简，但是蕴含着精华：

- 通过 Core Graphic 异步预渲染视图，在主线程中设置 layer 的 contents 属性。
- 给 runloop 增加 observer，在 runloop 即将结束的时候运行耗时逻辑，以防止阻塞主线程
- 创建处理器数量个串行队列作为 GCD 的执行队列，最大程度上利用多核 CPU 的优势。

## 思考题

1. 为什么会产生卡顿？
2. CPU 和 GPU 的职责有哪些？
3. CPU 方面如何优化？
4. GPU 如何优化？
5. YYAsyncLayer 为什么可以异步渲染？
6. YYAsyncLayer 在何时执行渲染？
7. YYAsyncLayer 如何通过串行队列实现异步的？

## 参考

[iOS 保持界面流畅的技巧](<https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/>)啥也不说了，经典中的经典

[YYAsyncLayer 源码剖析：异步绘制](<https://www.jianshu.com/p/154451e4bd42>)

[使用 ASDK 性能调优 - 提升 iOS 界面的渲染性能](<https://www.jianshu.com/p/0c187818b39f>)

不得不说，indulge_in 和 Draveness 是我见过的唯二文章写的深刻又易懂的人了。佩服