title: 许多 iOS API 的使用方式(持续更新)
date: 2017/2/13 14:07:12  
categories: iOS
tags: 

	- 学习笔记
	- 持续更新

------

很多不常用的简单的 API 的使用方式。分开写的话太短小，就合在一起吧。（这里都是学到的时候，东找一点西找一点拼凑起来的，一开始没有记录出处在哪，如果哪里有侵权的地方，还请尽快告知呀。我会第一时间注明出处的。谢谢啦~）

<!--more-->

## UIVisualEffectView 实现高斯模糊

如果想要给一个 view  添加一个高斯模糊的效果，只要在那个 view 上添加一个 `UIVisualEffectView` 即可。

高斯模糊有三种效果，从浅入深的 style 依次是：

- UIBlurEffectStyleExtraLight
- UIBlurEffectStyleLight
- UIBlurEffectStyleDark



使用的基本实例：

```objc
// 要添加模糊的view
UIImageView *imageview = [[UIImageView alloc] init];
imageview.frame = CGRectMake(10, 100, 300, 300);
imageview.image = [UIImage imageNamed:@"2"];
imageview.contentMode = UIViewContentModeScaleAspectFit;
imageview.userInteractionEnabled = YES;
[self.view addSubview:imageview];

// 高斯模糊的view
UIBlurEffect *blur = [UIBlurEffect effectWithStyle:UIBlurEffectStyleLight];
UIVisualEffectView *effectview = [[UIVisualEffectView alloc] initWithEffect:blur];
effectview.frame = CGRectMake(0, 0, imageview.size.width/2, 300);

// 添加高斯模糊
[imageview addSubview:effectview];
```



## UIInterpolatingMotionEffect 视图运动

`UIInterpolatingMotionEffect` 可以通过陀螺仪监测手机的倾斜情况。我们可以通过它设置视图响应的运动。

其中有两个属性：`minimumRelativeValue` 和 `maximumRelativeValue`。这两个属性控制图像运动的最大范围。

实例：

```objc
CGFloat effectOffset = 100.f;
// 设置x方向上的移动量
UIInterpolatingMotionEffect *effectX = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.x" type:UIInterpolatingMotionEffectTypeTiltAlongHorizontalAxis];
effectX.maximumRelativeValue = @(effectOffset);
effectX.minimumRelativeValue = @(-effectOffset);

// 设置y方向上的移动量
UIInterpolatingMotionEffect *effectY = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.y" type:UIInterpolatingMotionEffectTypeTiltAlongVerticalAxis];
effectY.maximumRelativeValue = @(effectOffset);
effectY.minimumRelativeValue = @(-effectOffset);

// 将移动量添加到数组中
UIMotionEffectGroup *group = [[UIMotionEffectGroup alloc] init];
group.motionEffects = @[effectX, effectY];

// 设置给 view
[self.view addMotionEffect:group];
```



## CADisplayLink 的使用

### 基本用法

`CADisplayLink` 是一个能让我们**以和屏幕刷新率同步的频率将特定的内容画到屏幕上的定时器类**。` CADisplayLink` 以特定模式注册到 runloop 后， 每当屏幕显示内容刷新结束的时候， runloop 就会向 `CADisplayLink` 指定的 target 发送一次指定的 selector 消息，  `CADisplayLink` 类对应的 selector 就会被调用一次。 

使用方式：

```objc
- (void)startDisplayLink
{
    self.displayLink = [CADisplayLink displayLinkWithTarget:self
                                                   selector:@selector(handleDisplayLink:)];
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop]
                           forMode:NSDefaultRunLoopMode];
}

- (void)handleDisplayLink:(CADisplayLink *)displayLink
{
    //do something
}

- (void)stopDisplayLink
{
    [self.displayLink invalidate];
    self.displayLink = nil;
}
```

当把 `CADisplayLink` 对象 add 到 runloop 中后（一定要加啊，否则无效），selector就能被周期性调用，类似于 `NSTimer` 被启动了；执行 invalidate 操作时， `CADisplayLink` 对象就会从 runloop 中移除，selector 调用也随即停止，类似于 `NSTimer` 的 invalidate 方法。

iOS 设备的 FPS 是60Hz，因此 `CADisplayLink` 的 selector 默认调用周期是每秒60次，这个周期可以通过 `frameInterval` 属性设置， `CADisplayLink`的 selector 每秒调用次数为 `60/ frameInterval`。比如当 `frameInterval` 设为2，每秒调用就变成30次。因此， `CADisplayLink` 周期的设置方式略显不便。不过 `CADisplayLink` 适合于需要精度较高的定时。

### 与 NSTimer 不同

**iOS**设备的屏幕刷新频率是固定的，`CADisplayLink`在正常情况下会在每次刷新结束都被调用，精确度相当高。
`NSTimer`的精确度就显得低了点，比如`NSTimer`的触发时间到的时候，`runloop`如果在阻塞状态，触发时间就会推迟到下一个`runloop`周期。并且 `NSTimer`新增了`tolerance`属性，让用户可以设置可以容忍的触发的时间的延迟范围。
`CADisplayLink`使用场合相对专一，适合做UI的不停重绘，比如自定义动画引擎或者视频播放的渲染。`NSTimer`的使用范围要广泛的多，各种需要单次或者循环定时处理的任务都可以使用。

> 可以通过 `CADisplayLink` 和上面的绘图，完成很多动画效果

