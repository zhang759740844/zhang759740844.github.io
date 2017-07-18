title: iOS 绘图
date: 2017/6/10 14:07:12  
categories: iOS
tags: 
	- UI
------

有些时候需求上的动画效果不能仅通过贴图来解决，这就需要我们自定义各种图案，然后动态的绘制完成动态效果。比较复杂，但是实现出来的东西非常精致。

<!--more-->


## UIBezierPath 的使用

### 绘制矩形

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

### **圆和椭圆**

替换上面的代码为：

```objc
// 圆
UIBezierPath* aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20, 20, 100, 100)];
// 椭圆
UIBezierPath* aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20, 20, 100, 50)];
```

都是需要一个外接矩形，椭圆和圆不同之处在于圆的外接矩形是个正方形。

### 多边形

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

### 不规则形状

要用弧线组成不规则的形状，我们需要用到中心点、弧度和半径。弧度以顺时针为准，0° 指向右边。

#### 绘制一段弧度

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

#### 贝塞尔曲线

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

## CoreGraphics 的使用
### CGContextRef 的使用

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

### 绘制矩形

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

### 圆和椭圆

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

### 多边形

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

### 不规则形状

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

### CGPath的使用
`CGPath` 和 `CGContextRef` 一样，同属于 CoreGraphics：
- CGContextRef：表示绘画上下文，也就是Quartz 2D 绘画的对象。
- CGPathRef：用数学来描绘一系列形状和线条的图形保存路径。

使用方式和上面类似，首先创建一个 `CGMutablePathRef`（属于CGPath类型）对象。接着通过 `CGPathAddEllipseInRect` 和 `CGPathAddArc` 等方法创建线条。可以把 `CGAffineTransform` 的地址传进去，这样 Transform 才会应用.接着把这个创建好的 CGPath 加入到当前 `CGContextRef` 中，最后通过 `CGContextRef` 执行绘画。这里是一个笑脸的示意：
```objc
//获取当前CGContextRef
CGContextRef gc = UIGraphicsGetCurrentContext();
    
//创建用于转移坐标的Transform，这样我们不用按照实际显示做坐标计算
// 就表示以 50，50 为起点
CGAffineTransform transform = CGAffineTransformMakeTranslation(50, 50);
//创建CGMutablePathRef
CGMutablePathRef path = CGPathCreateMutable();
//左眼
CGPathAddEllipseInRect(path, &transform, CGRectMake(0, 0, 20, 20));
//右眼
CGPathAddEllipseInRect(path, &transform, CGRectMake(80, 0, 20, 20));
//笑
CGPathMoveToPoint(path, &transform, 100, 50);
CGPathAddArc(path, &transform, 50, 50, 50, 0, M_PI, NO);
//将CGMutablePathRef添加到当前Context内
CGContextAddPath(gc, path);
//设置绘图属性
[[UIColor blueColor] setStroke];
CGContextSetLineWidth(gc, 2);
//执行绘画
CGContextStrokePath(gc);
```

### 添加阴影效果

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



## CAShapeLayer 的使用

上面都是静态的一些图形，绘制完成后就不能再被更改，可以将其放在某个 View 的 `drawRect:` 方法中，但是这样每次都会重新创建一个新的图形，如何保存下图形对象并且修改，可以通过 CAShapeLayer 实现。CAShapeLayer 是 CALayer 的子类，但是比 CALayer 更灵活，可以画出各种图形。

CAShapeLayer 有一个神奇的属性 `path` 用这个属性配合上 UIBezierPath 这个类（以及上面说的的 CGPath 也行）就可以达到超神的效果。

```swift
let path = UIBezierPath(rect: CGRectMake(a, b, c, d))
let layer = CAShapeLayer()
layer.path = path.CGPath
layer.fillColor = UIColor.blackColor().CGColor
view.layer.addSublayer(layer)
```

这样，我们就可以把 layer 保存下来。如果我们需要动态改变 layer 的时候（可以搭配CADisplayLink使用），比方要把 a 改成另外一个值，那么直接修改 a 的值，然后调用 `[layer setNeedsDisplay]` 即可。

CAShapeLayer 还可以设置动画属性控制图形的显示过程（即从头还是从尾还是从中间显示）





