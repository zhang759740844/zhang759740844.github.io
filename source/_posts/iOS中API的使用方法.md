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



## 绘制图形

### UIBezierPath 的使用

#### 绘制矩形

```objc
- (void)drawRect:(CGRect)rect
{
    UIColor *color = [UIColor colorWithRed:0 green:0 blue:0.7 alpha:1];
    [color set]; //设置线条颜色

    UIBezierPath* aPath = [UIBezierPath bezierPathWithRect:CGRectMake(20, 20, 100, 50)];
    aPath.lineWidth = 8.0;
    aPath.lineCapStyle = kCGLineCapRound; //线条拐角
    aPath.lineJoinStyle = kCGLineCapRound; //终点处理

    [aPath stroke];
}
```

#### **圆和椭圆**

替换上面的代码为：

```objc
// 圆
UIBezierPath* aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20, 20, 100, 100)];
// 椭圆
UIBezierPath* aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20, 20, 100, 50)];
```

都是需要一个外接矩形，椭圆和圆不同之处在于圆的外接矩形是个正方形。

#### 多边形

多边形是一些简单的形状,这些形状是由一些直线线条组成，我们可以用 `moveToPoint:` 和 `addLineToPoint:` 方法去构建。`moveToPoint:` 设置我们想要创建形状的起点。从这点开始，我们可以用方法 `addLineToPoint:` 去创建一个形状的线段。我们可以连续的创建 line，每一个 line 的起点都是先前的终点，终点就是指定的点。`closePath` 可以在最后一个点和第一个点之间画一条线段。

```objc
- (void)drawRect:(CGRect)rect
{
    UIColor *color = [UIColor colorWithRed:0 green:0.7 blue:0 alpha:1];
    [color set];

    UIBezierPath* aPath = [UIBezierPath bezierPath];
    aPath.lineWidth = 5.0;

    aPath.lineCapStyle = kCGLineCapRound;
    aPath.lineJoinStyle = kCGLineCapRound;

    // 起点
    [aPath moveToPoint:CGPointMake(100.0, 0.0)];

    // 绘制线条
    [aPath addLineToPoint:CGPointMake(200.0, 40.0)];
    [aPath addLineToPoint:CGPointMake(160, 140)];
    [aPath addLineToPoint:CGPointMake(40.0, 140)];
    [aPath addLineToPoint:CGPointMake(0.0, 40.0)];
    [aPath closePath];//第五条线通过调用closePath方法得到的

    //根据坐标点连线
    [aPath stroke];
  	// 将图形填充满
    [aPath fill];
}
```

#### 不规则形状

要用弧线组成不规则的形状，我们需要用到中心点、弧度和半径。弧度以顺时针为准，0° 指向右边。

##### 绘制一段弧度

```objc
- (void)drawRect:(CGRect)rect
{
    UIColor *color = [UIColor redColor];
    [color set]; //设置线条颜色

    UIBezierPath* aPath = [UIBezierPath bezierPathWithArcCenter:CGPointMake(80, 80)
                                                         radius:75
                                                     startAngle:0
                                                       endAngle:DEGREES_TO_RADIANS(135)
                                                      clockwise:YES];

    aPath.lineWidth = 5.0;
    aPath.lineCapStyle = kCGLineCapRound; //线条拐角
    aPath.lineJoinStyle = kCGLineCapRound; //终点处理

    [aPath stroke];
}
```

##### 贝塞尔曲线

贝塞尔曲线需要一个起始点，终点和控制点

![bezier曲线](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/bezier1.png?raw=true)

```objc
- (void)drawRect:(CGRect)rect
{
    UIColor *color = [UIColor redColor];
    [color set]; //设置线条颜色

    UIBezierPath* aPath = [UIBezierPath bezierPath];

    aPath.lineWidth = 5.0;
    aPath.lineCapStyle = kCGLineCapRound; //线条拐角
    aPath.lineJoinStyle = kCGLineCapRound; //终点处理
	// 起始点
    [aPath moveToPoint:CGPointMake(20, 100)];
	// 终点 和 控制点
    [aPath addQuadCurveToPoint:CGPointMake(120, 100) controlPoint:CGPointMake(70, 0)];

    [aPath stroke];
}
```

贝塞尔曲线可以有多个控制点，可以实现类似波浪的效果：

```objc
- (void)drawRect:(CGRect)rect
{
    UIColor *color = [UIColor redColor];
    [color set]; //设置线条颜色

    UIBezierPath* aPath = [UIBezierPath bezierPath];

    aPath.lineWidth = 5.0;
    aPath.lineCapStyle = kCGLineCapRound; //线条拐角
    aPath.lineJoinStyle = kCGLineCapRound; //终点处理

    [aPath moveToPoint:CGPointMake(5, 80)];

    [aPath addCurveToPoint:CGPointMake(155, 80) controlPoint1:CGPointMake(80, 0) controlPoint2:CGPointMake(110, 100)];

    [aPath stroke];
}
```

### CoreGraphics 的使用

使用中，`UIBezierPath` 是 `CoreGraphics` 的封装，使用它可以完成大部分的绘图操作，不过更底层的 `CoreGraphics` 更加强大。

由于像素是依赖于目标的，所以2D绘图并不能操作单独的像素，我们可以从上下文（Context）读取它。所以我们在绘制之前需要通过:

```objc
CGContextRef ctx = UIGraphicsGetCurrentContext()
```

获取当前推入堆栈的图形，相当于你所要绘制图形的图纸，然后绘图就好比在画布上拿着画笔机械的进行画画，通过制定不同的参数来进行不同的绘制。

画完之后我们需要通过下面方法完成最后的绘制。

```objc
CGContextSetFillColorWithColor(CGContextRef c, CGColorRef color)
CGContextFillPath(CGContextRef c)
```

#### 绘制矩形

绘制矩形需要先定义矩形的 rect，然后使用下面方法进行绘制：

```objc
CGContextAddRect(CGContextRef c, CGRect rect)
```

完整的方法如下：

```objc
- (void)drawRectangle {
    CGRect rectangle = CGRectMake(80, 400, 160, 60);
    CGContextRef ctx = UIGraphicsGetCurrentContext();

    CGContextAddRect(ctx, rectangle);
  
    CGContextSetFillColorWithColor(ctx, [UIColor lightGrayColor].CGColor);
    CGContextFillPath(ctx);
}
```

这样就绘制出一个灰色的矩形。

#### 圆和椭圆

绘制圆和绘制弧形是一个方法：

```objc
CGContextAddArc(CGContextRef c, CGFloat x, CGFloat y, CGFloat radius, CGFloat startAngle, CGFloat endAngle, int clockwise)
```

参数说明如下：

```objc
c           当前图形
x           中心点横坐标x
y           中心点纵坐标y
radius      半径
startAngle  弧的起点与正X轴的夹角
endAngle    弧的终点与正X轴的夹角
clockwise   指定1创建一个顺时针的圆弧，或是指定0创建一个逆时针圆弧
```

创建一个圆形方法如下：

```objc
- (void)drawCircleAtX:(float)x Y:(float)y {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGContextAddArc(ctx, x, y, 150, 0, 2 * M_PI, 1);
    CGContextSetFillColorWithColor(ctx, [UIColor blackColor].CGColor);
    CGContextFillPath(ctx);
}
```

绘制椭圆需要先设置一个容纳椭圆的矩形：

```objc
CGContextAddEllipseInRect(CGContextRef context, CGRect rect)
```

完整方法如下：

```objc
- (void)drawEllipseAtX:(float)x Y:(float)y {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGRect rectangle = CGRectMake(x, y, 60, 30);
    CGContextAddEllipseInRect(ctx, rectangle);
    CGContextSetFillColorWithColor(ctx, [UIColor redColor].CGColor);
    CGContextFillPath(ctx);
}
```

#### 多边形

其实和 `UIBezierPath` 的方法名都很像，只是 `UIBezierPath` 是面向对象的，这个是面向过程的。

绘制多边形需要通过 `CGContextMoveToPoint` 从一个开始点开始一个新的子路径，然后通过 `CGContextAddLineToPoint` 在当前点追加直线段，最后通过 ``CGContextClosePath` 关闭路径即可。如下我们绘制一个三角形：

```objc
- (void)drawTriangle {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGContextBeginPath(ctx);

    CGContextMoveToPoint(ctx, 160, 40);
    CGContextAddLineToPoint(ctx, 190, 80);
    CGContextAddLineToPoint(ctx, 130, 80);
    CGContextClosePath(ctx);

    CGContextSetFillColorWithColor(ctx, [UIColor blackColor].CGColor);
    CGContextFillPath(ctx);
}
```

#### 不规则形状

这里再重新绘制一下上面介绍的两种贝塞尔曲线，首先需要一个给定的起始点：

```objc
CGContextMoveToPoint(CGContextRef c, CGFloat x, CGFloat y)
```

再给定控制点和终点：

```objc
CGContextAddQuadCurveToPoint(CGContextRef c, CGFloat cpx, CGFloat cpy, CGFloat x, CGFloat y)
```

其中命名关于 `cp` 的表示控制点坐标。

完整的代码如下：

```objc
- (void)drawQuadCurve {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGContextBeginPath(ctx);

    CGContextMoveToPoint(ctx, 50, 130);
    CGContextAddQuadCurveToPoint(ctx, 0, 100, 25, 170);

    CGContextSetLineWidth(ctx, 10);
    CGContextSetStrokeColorWithColor(ctx, [UIColor brownColor].CGColor);
    CGContextStrokePath(ctx);
}
```

给定两个控制点的方法名和上述的一样。只是多了两个坐标输入变量。

#### 添加阴影效果

添加阴影效果很简单，只要在要添加的 view 中添加相应方法即可：

```objc
CGContextSetShadowWithColor(CGContextRef context, CGSize offset, CGFloat blur, CGColorRef color)
```

设置阴影效果，4个参数分别是图形上下文，偏移量（CGSize），模糊值，和阴影颜色。下面看完整的例子：

```objc
- (void)drawCircleAtX:(float)x Y:(float)y {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGContextAddArc(ctx, x, y, 150, 0, 2 * M_PI, 1);
    CGContextSetShadowWithColor(ctx, CGSizeMake(10, 10), 20.0f, [[UIColor grayColor] CGColor]);
    CGContextSetFillColorWithColor(ctx, [UIColor yellowColor].CGColor);
    CGContextFillPath(ctx);
}
```



## CADisplayLink 方式的定时器

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

当把 `CADisplayLink` 对象 add 到 runloop 中后，selector就能被周期性调用，类似于 `NSTimer` 被启动了；执行 invalidate 操作时， `CADisplayLink` 对象就会从 runloop 中移除，selector 调用也随即停止，类似于 `NSTimer` 的 invalidate 方法。

iOS 设备的 FPS 是60Hz，因此 `CADisplayLink` 的 selector 默认调用周期是每秒60次，这个周期可以通过 `frameInterval` 属性设置， `CADisplayLink`的 selector 每秒调用次数为 `60/ frameInterval`。比如当 `frameInterval` 设为2，每秒调用就变成30次。因此， `CADisplayLink` 周期的设置方式略显不便。不过 `CADisplayLink` 适合于需要精度较高的定时。





