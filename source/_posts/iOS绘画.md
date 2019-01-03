title: iOS 绘图
date: 2017/6/10 14:07:12  
categories: iOS
tags: 
	- UI
------

有些时候需求上的动画效果不能仅通过贴图来解决，这就需要我们自定义各种图案，然后动态的绘制完成动态效果。比较复杂，但是实现出来的东西非常精致。

<!--more-->


## drawRect 获取当前图形上下文绘图

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

> closePath 可以将路径封闭

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

### 根据手势绘图

方法是根据手势移动生成相应的贝塞尔曲线，然后设置重绘。在 `drawRect` 中把这些曲线绘制出来：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/drawrect_1.png?raw=true)

### stroke 方法的解释

上文中都是在 `drawRect` 方法中通过 `UIBezierPath` 绘制的。`stroke` 实质上帮我们省略了好几个步骤，包括：

1. 获取当前上下文：`CGContextRef ctx = UIGraphicsGetCurrentContext()`
2. 把贝塞尔曲线添加到上下文中：`CGContextAddPath(ctx, aPath.CGPath)` 
3. 绘制图形：`CGContextStrokePath(ctx)`

也就是说在 `drawRect` 中都是需要开启上下文的。只不过通过 `stroke` 方式就省略了。

###  drawRect 的问题

drawRect 会生成一个图形上下文，这个空间为 `图层宽*图层高*4 字节`，会引起内存的暴增。所以，一般情况下，不要使用 drawRect 方法。

CAShapeLayer是一个通过矢量图形而不是bitmap来绘制的图层子类。用CGPath来定义想要绘制的图形，它的优点有:
	1. 渲染快，有 GPU 硬件加速
	2. 不会占用系统内存
	3. 不会被图层边界裁减掉
	4. 不会有像素化

推荐使用 CAShapeLayer，使用方式差不多，drawRect 是把 path add 到 context 上，CAShapeLayer 是把 path 赋给 layer.path

## 创建新的图形上下文绘图

上面 `drawRect` 使用的是当前图形上下文`UIGraphicsGetCurrentContext()` 绘制的。事实上，可以在任何时候，创建新的图形上下文新建 UIImage

### 在图片上写字生成新的图片

生成一个位图上下文，然后把图片和文字画上去，再获取位图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/drawrect_2.png?raw=true)

### 裁剪图片（这个方法可以用来绘制圆角图片）

先设置裁剪的范围为一个贝塞尔曲线，然后把图片画到贝塞尔曲线中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/drawrect_3.png?raw=true)

### 生成一张带边框的圆角图片

这在上面的基础上添加边框，也就是改变绘图的位置：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/drawrect_4?raw=true)

### 截图

开启一个上下文，然后把视图通过 `renderInContext:` 绘制到上下文上。这种方式获取的是 UIImage 对象：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/drawrect_5.png?raw=true) 

当然还有一种情况，直接通过系统 api 获取，该方法生成一个 UIView：

```objc
- (UIView *)snapshotView {
    UIView *snapView = [self snapshotViewAfterScreenUpdates:YES];
    return snapView;
}
```

## CAShapeLayer 的使用

只要把贝塞尔曲线的 CGPath 赋值给 CAShapeLayer 的 path 就可以了，不需要调用任何重绘方法。它的基本属性有：

- path：绘制路径
- fillColor：填充颜色
- fillRule：填充规则，就是如果图形里面也有线条，那么是否中间也要填充
- strokeColor：描边的颜色
- strokeStart：描边的开始点，默认为0最大为1
- strokeEnd：描边的结束点，默认为1 (**配合核心动画可以实现那种慢慢变化的效果**)
- lineWidth：线宽
- lineCap：线的端点的样式
- lineJoin：线交界处样式

### 使用 CAShapeLayer 设置圆角

将 CAShapeLayer 作为图片的 layer 的mask：
```objc
let roundedRectPath = UIBezierPath(roundedRect: avatorView.bounds, byRoundingCorners: .AllCorners, cornerRadii: CGSize(width: 10, height: 10))
let shapeLayer = CAShapeLayer()
shapeLayer.path = roundedRectPath.CGPath
avatorView.layer.mask = shapeLayer
```
> 这种通过 mask 的方式不会消除离屏渲染，只是这种方式比 cornerRadius 性能好一些。

### 进度条

不断的给 layer 赋不断边长的贝塞尔曲线

```objc
- (void)viewDidLoad {
    // 创建一个 layer
    CAShapeLayer *layer = [CAShapeLayer layer];
    layer.bounds = CGRectMake(0, 0, 200, 45);
    layer.position = self.view.center;
    layer.path = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, count / 6 * 2, 45)].CGPath;
    layer.fillColor = [UIColor redColor].CGColor;
    layer.fillRule = kCAFillRuleEvenOdd;
    self.redLayer = layer;
    // 添加 layer 到图层上
    [self.view.layer addSublayer:self.redLayer];
    // 添加一个定时器
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(action)];
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
}

// 定时器方法
- (void)action {
    count ++;
    self.redLayer.path = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, count / 6 * 2, 45)].CGPath;
    
    if (count > 60 * 10 -1) {
        [self.displayLink invalidate];
    }
}
```

> 图层的 bounds 的 x，y 不能瞎改，否则 path 会跟着 x，y 变化

### View 消除阴影的离屏渲染
准确的说，这不属于 CAShapeLayer。

如果直接对一个 View 加一个 shadow，那么会产生离屏渲染。我们可以指定 `shaowPath`，避免离屏渲染：

```objc
let imageViewLayer = avatorView.layer
imageViewLayer.shadowColor = UIColor.blackColor().CGColor
imageViewLayer.shadowOpacity = 1.0 //此参数默认为0，即阴影不显示
imageViewLayer.shadowRadius = 2.0 //给阴影加上圆角，对性能无明显影响
imageViewLayer.shadowOffset = CGSize(width: 5, height: 5)
//设定路径：与视图的边界相同
let path = UIBezierPath(rect: cell.imageView.bounds)
imageViewLayer.shadowPath = path.CGPath//路径默认为 nil
```

