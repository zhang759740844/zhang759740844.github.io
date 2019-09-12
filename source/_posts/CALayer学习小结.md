title: CALayer学习小结
date: 2016/8/12 14:07:12  
categories: iOS
tags: 

	- Animation
	- UI

---

这两天想大致学习下animation的使用方法。看了[文顶顶的ios开发UI篇](http://www.cnblogs.com/wendingding/tag/UI高级/)专题，写的很好，学习了很多。再摘录部分，以作备忘。

<!--more-->

## CALayer简介
### 简介
UIView之所以能显示在屏幕上，因为内部的layer。创建UIView的时候自动创建CALayer对象。UIView本身不具备显示的功能，拥有显示功能的是它内部的图层。

### 基本属性
通过操作这个CALayer对象，可以很方便地调整UIView的一些界面属性，比如：阴影、圆角大小、边框宽度和颜色等。
```objc
//设置边框的宽度为20
self.customView.layer.borderWidth=20;
//设置边框的颜色(borderColor是CGColor类型)
self.customView.layer.borderColor=[UIColor greenColor].CGColor;

//设置layer的圆角
self.customView.layer.cornerRadius=20;
//设置超过子图层的部分裁减掉
self.iconView.layer.masksToBounds=YES;

//在view的图层上添加一个image，contents表示接受内容
self.customView.layer.contents=(id)[UIImage imageNamed:@"cat"].CGImage;

//设置阴影的颜色
self.customView.layer.shadowColor=[UIColor blackColor].CGColor;
//设置阴影的偏移量，如果为正数，则代表为往右边偏移
self.customView.layer.shadowOffset=CGSizeMake(15, 5);
//设置阴影的透明度(0~1之间，0表示完全透明)
self.customView.layer.shadowOpacity=0.6;

//通过uiview设置（2D效果）
self.customView.transform=CGAffineTransformMakeTranslation(0, -100);
//通过layer来设置（3D效果,x，y，z三个方向）
self.iconView.layer.transform=CATransform3DMakeTranslation(100, 20, 0);
//旋转
self.iconView.layer.transform=CATransform3DMakeRotation(M_PI_4, 1, 1, 0.5);
```

> 注意，这里的颜色不是 UIColor 类型，而是 CGColor 类型。需要使用 [UIColor redColor].CGColor 的方式获取，而不是使用强制转型把 UIColor 转为 CGColor。

### position，anchorPoint，bounds 以及 frame 的概念

- frame: 用来描述自己在父视图的位置，即 x y 是相对于父视图的起点的。 
- bounds: 描述当前视图的左上角的相对子视图的坐标，以及视图大小的。一般 `bounds.origin` 为 (0, 0) ，如果把它改为 (-50, 0)，就表示左上角的坐标为  (-50, 0)，现在子视图相对的原点坐标就在左上角右边 50 的地方。所以相当于把所有子视图向右移动 50.
- position:设置CALayer在父层中的位置，这个位置会和锚点重合。
- anchorPoint:决定着CALayer身上的哪个点会在position属性所指的位置。以自己的左上角为原点(0, 0)，它的x、y取值范围都是0~1，**默认值为（0.5, 0.5）**。**layer 的缩放和旋转，都是以 anchorPoint 为原点的**。

position，anchorpoint 和 bounds 共同决定了视图的 frame。一般情况下，frame 的大小等于 bounds 的大小，但是如果视图旋转了，frame 会变为旋转视图的外界矩形的大小。

> 如果我们想移动一个视图的所有子视图，可以修改 `bonuds.orgin`。不过我们也可以在视图上添加一个 `contentView `，然后移动它的 frame。

### 隐式动画

每个View内部都关联着一个Root Layer。所有非rootlayer都存在着隐式动画。隐式动画就是对于layer的部分属性进行修改时，默认会产生的一些动画。

常见的动画属性：

- bounds：用于设置CALayer的宽度和高度。修改这个属性会产生缩放动画
- backgroundColor：用于设置CALayer的背景色。修改这个属性会产生背景色的渐变动画
- position：用于设置CALayer的位置。修改这个属性会产生平移动画

### 创建图层
```objc
- (void)viewDidLoad
{
    [super viewDidLoad];
    //创建一个layer
    CALayer *Mylayer=[CALayer layer];
    //设置layer的属性
    Mylayer.bounds=CGRectMake(100, 100, 100, 100);
    Mylayer.position=CGPointMake(100, 100);
    
    //设置需要显示的图片
    Mylayer.contents=(id)[UIImage imageNamed:@"me"].CGImage;
    //设置圆角半径为10
    Mylayer.cornerRadius=10;
    //如果设置了图片，那么需要设置这个属性为YES才能显示圆角效果
    Mylayer.masksToBounds=YES;
    //设置边框
    Mylayer.borderWidth=3;
    Mylayer.borderColor=[UIColor brownColor].CGColor;
    
    //把layer添加到界面上
    [self.view.layer addSublayer:Mylayer];
}
```

### 总结
对比CALayer，UIView多了一个事件处理的功能。也就是说，CALayer不能处理用户的触摸事件，而UIView可以。
如果显示出来的东西需要跟用户进行交互的话，用UIView；如果不需要跟用户进行交互，用UIView或者CALayer都可以

## UIView的transform属性

transform是view的一个重要属性,它在矩阵层面上改变view的显⽰状态,能实现view的缩放、旋转、平移等功能。transform是`CGAffineTransform`类型的。

### transform结构

transform是一个`CGAffineTransform`类型，结构如下：

```objc
struct CGAffineTransform {
  CGFloat a, b, c, d;
  CGFloat tx, ty;
};
```

CGAffineTransform实际上是一个矩阵

```objc
| a,  b,  0 |
| c,  d,  0 |
| tx, ty, 1 |
```

由于transform只有两维，需要一个3阶矩阵来表示其缩放以及平移的变化。

坐标变换过程：

```objc
                    | a,  b,  0 |
{x',y',1}={x,y,1} x | c,  d,  0 |
                    | tx, ty, 1 |
                    
==>

xn=ax+cy+tx;
yn=bx+dy+ty;

```

这个矩阵的第三列是固定的，所以每次变换时，只需传入前两列的六个参数[a,b,c,d,tx,ty]即可。

### transform方法

在`CGAffineTransform`的生成函数中，大多是两两对应的，一个带
make字样，一个不带。带make字样的是直接生成一个新的`CGAffineTransform`，不带make字样的则是在一个`CGAffineTransform`的基础上生成新的。函数返回值均是`CGAffineTransform`类型。

多个`CGAffineTransform`对象赋给view，最终只执行最后一个动画，多个动画需要组合在一起。

#### scale

实现的是放大和缩小:

```objc
CGAffineTransformScale(CGAffineTransform t,
  CGFloat sx, CGFloat sy)；
CGAffineTransformMakeScale(CGFloat sx, CGFloat sy)；
```

生成新的transform相当于将`t' = [sx ，0 ，0，sy ，0， 0]`这六个参数代入矩阵中,即改变a和d。

#### rotate

实现的是旋转：

```objc
CGAffineTransformRotate(CGAffineTransform t,
  CGFloat angle)
CGAffineTransformMakeRotation(CGFloat angle)；
```

angle为角度，angle=π则旋转180度。矩阵的六个参数为`t' =  [ cos(angle)，sin(angle)，-sin(angle)，cos(angle) 0，0]；`

#### translate

实现的是平移：

```objc
CGAffineTransformTranslate(CGAffineTransform t,
  CGFloat tx, CGFloat ty)；
CGAffineTransformMakeTranslation(CGFloat tx,
  CGFloat ty)；
```

矩阵的六个参数为`t' = [1，0，0，1，tx，ty] ；`代入公式，`xn=x+tx,yn=y+ty`

#### 复原

```objc
view.transform＝CGAffineTransformIdentity;
```

上述的各种动画的变化都是以原始图像为基准的，而不是在变化后继续变化。`CGAffineTransformIdentity`将view从当前状态复原回view最初始的状态。

> 一个技巧。如果使用 AutoLayout 的视图要添加动画，一般我们会把约束拖到代码中，然后在动画前修改约束，在动画的 block 中调用其父控件的 `layoutIfNeeded`。
>
> 还有一种更好的办法，在动画的 block 中直接用 view 的 transform 属性。

### 设置 transform 进行动画和 设置 frame 动画的区别

视图的 frame 其实是由视图的 bounds，position，anchorpoint 以及 transform 共同决定的。修改 frame 其实就是修改 bounds 和 position 的值。

一般而言，最好使用 transform 进行动画，因为有一种情况是，如果该视图具有子视图，并且要进行缩放动画。那么执行动画的时候，动画虽然有一个过程，但其实从动画一开始，frame就已经修改了。如果直接设置frame，那么开始的时候，子视图就会按变化后的frame来重新布局，而不是跟随父视图一起慢慢变化。

而 transform 动画的话，不会调用控件的 `layoutsubview` 方法，整个过程更像是把整个视图按比例缩放，子视图会跟着父视图一起缩放。

## CALayer的transform属性

### transform结构

CALayer的transform是一个`CATransform3D`结构：

```objc
struct CATransform3D
{
  CGFloat m11, m12, m13, m14;
  CGFloat m21, m22, m23, m24;
  CGFloat m31, m32, m33, m34;
  CGFloat m41, m42, m43, m44;
};
```

有别于`CGAffineTransform`,`CATransform3D`是一个三维变化，需要一个4阶矩阵表示。其他类似，再次不表。

### transform方法

CALayer的transform方法和View的transform基本一致。举几点不同：

- CALayer由于有z轴，因此对不同图层使用`CATransform3DMakeTranslation (CGFloat tx, CGFloat ty, CGFloat tz)`方法的时候可以通过改变tz的值，来实现图层的覆盖。对于tz来说，值越大，那么图层就越往外（接近屏幕），值越小，图层越往里（屏幕里）。
- 由于图像是从正面投影，直接绕着x或y轴旋转达不到透视的效果，如果要达到透视效果，需要改变`m34`(其实改变m14，m24也能达到效果，可以自行通过行列式推导。)，再对图层进行旋转。`m34`的值可以根据需要实现的效果推导得到，不直接的方法还是直接试。
- 如果想要直接改变矩阵里的值，可以先使用`CATransform3DIdentity`的方式，初始化一个`CATransform3D`实例，然后再赋值。

## CAShapeLayer

`CAShapeLayer`继承自CALayer。`CAShapeLayer`是在坐标系内绘制贝塞尔曲线的，通过绘制贝塞尔曲线，设置shape(形状)的path(路径)，从而绘制各种各样的图形以及不规则图形。因此，使用`CAShapeLayer`需要与`UIBezierPath`一起使用。

### 创建

```objc
CAShapeLayer *layer = [CAShapeLayer layer];
```

### 属性

```objc
// 要呈现的路径
@property(nullable) CGPathRef path;
// 填充色
@property(nullable) CGColorRef fillColor;
// 填充规则。值有两种，非零和奇偶数，但默认是非零值。
@property(copy) NSString *fillRule;
// 设置描边色
@property(nullable) CGColorRef strokeColor;
// 绘制边线轮廓路径的子区域。该值必须在[0,1]范围，0代表路径的开始，1代表路径的结束。在0和1之间的值沿路径长度进行线性插值。strokestart默认为0，strokeend默认为1。
@property CGFloat strokeStart;
@property CGFloat strokeEnd;
// 线的宽度
@property CGFloat lineWidth;
// 端点和交点的显示类型
@property(copy) NSString *lineCap;
@property(copy) NSString *lineJoin;

```

### 创建贝塞尔曲线

这里只是给了几个图形的基本画法，还有更多图形的画法可以稍后查看

```objc
//绘制矩形
UIBezierPath *path = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, 100, 100)];
//绘制圆形路径
UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, 100, 100)];
//绘制自带圆角的路径
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0, 0, 100, 100) cornerRadius:30];
//指定矩形某一个角加圆角（代码示例为左上角）
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0, 0, 100, 100) byRoundingCorners:UIRectCornerTopLeft cornerRadii:CGSizeMake(50, 50)];

self.layer.path = path.CGPath;
```

### 曲线动画

曲线动画的主要思想是对 layer 的 `strokeEnd` 属性做 `CABasicAnimation` 动画：

```objc
UIBezierPath *path = [UIBezierPath bezierPath];
//起始点
[path moveToPoint:CGPointMake(50, 667/2)];
//结束点、两个控制点
[path addCurveToPoint:CGPointMake(330, 667/2) controlPoint1:CGPointMake(125, 200) controlPoint2:CGPointMake(185, 450)];

CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
animation.duration = 5;
animation.fromValue = @(0);
animation.toValue = @(1);
animation.repeatCount = 100;

CAShapeLayer *layer = [self createShapeLayerNoFrame:[UIColor clearColor]];
layer.path = path.CGPath;
layer.lineWidth = 2.0;
[layer addAnimation:animation forKey:@"strokeEndAnimation"];
```





## Mask属性

### 基本使用

layer的大小和形状是受到mask遮罩层的影响的，可以通过赋给mask层一个新layer，来实现改变layer形状的效果。mask图层的 Color 属性是无关紧要的（**mask不是透明的部分，layer能显示出原来的颜色**），真正重要的是图层的轮廓。

下面的例子中，为一个图片设置了圆形的蒙版。蒙版外的部分是透明的，该部分图片不予显示。

```objc
- (CALayer *)maskRadiusCorner:(UIImageView *)imageView{
    //CAShapeLayer 是 CALayer 的子类，通过UIBezierPath来绘制它的形状
    CAShapeLayer *masklayer = [CAShapeLayer layer];
    //获取宽度
    masklayer.frame = imageView.bounds;
    //注意bezierPathWithArcCenter的设置
    masklayer.path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(imageView.frame.size.width/2, imageView.frame.size.height/2) radius:imageView.frame.size.width/2 startAngle:0 endAngle:2*M_PI clockwise:YES].CGPath;
    return masklayer;
}
```

效果图如下：

![sample](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mask_sample.jpeg?raw=true)

再举一个例子，观察下图的实现方式：

![sample](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mask_sample2.gif?raw=true)

原理是将彩色的图片添加在灰色的图片上，对彩色的图片添加一个圆形的蒙版。圆形蒙版外的部分由于是透明的所以就不予显示，也就是下面的灰色图片。圆形蒙版内的部分由于设置了颜色，就能够显示彩色图片。

```objc
/**
 添加一个圆形蒙版
 */
- (void)addMaskLayer{
    //创建蒙版的layer
    self.maskLayer = [CALayer layer];
    //蒙版大小
    self.maskLayer.frame = CGRectMake(0, 0, 100, 100);
    
    //随便取个颜色，只要不是透明的就行
    self.maskLayer.backgroundColor = [UIColor whiteColor].CGColor;
    //圆形蒙版
    self.maskLayer.cornerRadius = 50;
    //将蒙版赋给View
    self.colorImageView.layer.mask = self.maskLayer;
}

- (void)viewDidLoad{
    [super viewDidLoad];
    
    //添加两个image
    self.colorImageView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"pic"]];
    self.grayImageView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"gray"]];
    //设置frame
    self.colorImageView.frame = CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height);
    self.grayImageView.frame = CGRectMake(0, 0, 300, 300);
    
    [self.view addSubview:self.colorImageView];
    [self.view addSubview:self.grayImageView];
    
    [self addMaskLayer];
}
```

### 绘制只有两个圆角的视图

有些情况下，一个 button 或者 label，只要右边的两个角圆角，或者只要一个圆角。该怎么办呢？

```objc
CGRect rect = CGRectMake(0, 0, 100, 50);
CGSize radio = CGSizeMake(5, 5); // 圆角尺寸
UIRectCorner corner = UIRectCornerTopLeft | UIRectCornerTopRight; // 这只圆角位置
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corner cornerRadii:radio];
CAShapeLayer *masklayer = [[CAShapeLayer alloc] init]; // 创建shapelayer
masklayer.frame = button.bounds;
masklayer.path = path.CGPath; // 设置路径
button.layer.mask = masklayer;
```

## 关于离屏渲染

### 渲染机制

CPU 计算内容交由 GPU 渲染，GPU 渲染完成后放入帧缓冲区。随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。

### GPU 渲染方式

- On-Screen Rendering：意为当前屏幕渲染。GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行。
- Off-Screen Rendering：意为离屏渲染。GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。

其实非当前屏幕缓冲区的渲染都叫做离屏渲染，CPU 渲染得到 bitmap 然后交由 GPU 显示也是一种**特殊的离屏渲染**，此举会消耗一定的 CPU 资源。但是如果 GPU 资源紧张，同时 CPU 空闲，可以考虑如此优化，以消除 GPU 不能及时渲染导致的丢帧的影响。

所以离屏渲染不一定就是影响性能的。只是渲染方式的一种。

### 离屏渲染的触发

- shouldRasterize（光栅化）
- masks（遮罩）
- shadows（阴影）
- edge antialiasing（抗锯齿）
- group opacity（不透明）

其中，光栅化是把GPU的操作转到CPU上，严格意义上是"软件渲染"，将图转化为一个个栅格组成的图象。并且缓存起来，减少渲染的频度。对于基本不会变化的视图，开启光栅化有助于性能优化，但是如果内容经常变化，那么就会造成性能浪费。

### 为什么会产生离屏渲染

当使用圆角，阴影，遮罩的时候，图层属性的混合体被指定为在未预合成之前不能直接在屏幕中绘制。

**这是因为 GPU 无法在某一层渲染完成之后，再回过头来擦除/改变其中的某个部分。因为在这一层之前的若干层layer像素数据，已经在渲染中被永久覆盖了。这就意味着，对于每一层layer，要么能找到一种通过单次遍历就能完成渲染的算法，要么就不得不另开一块内存，借助这个临时中转区域来完成一些更复杂的、多次的修改/剪裁操作。**

### 视图圆角

设置视图的 `cornerRadius`本身并不会触发离屏渲染，需要和 `maskToBounds` 一起设置才会产生。

一般情况下，直接设置圆角即可。

```objc
UIView *view = [[UIView alloc] init];
view.backgroundColor = [UIColor blackColor];
view.layer.cornerRadius = 3.f;
```

但是 UILabel 例外，UILabel 如果有背景色需要设置 layer 的背景色：

```objc
UILabel *label = [[UILabel alloc] init];
// 重点在此！！设置视图的图层背景色，千万不要直接设置 label.backgroundColor
label.layer.backgroundColor = [UIColor grayColor].CGColor;
label.layer.cornerRadius = cornerRadius;
```

UIImageView 无法做到直接隐藏图片圆角。所以需要自己绘制出一个带圆角的图片：

```swift
extension UIImage {  
    func kt_drawRectWithRoundedCorner(radius radius: CGFloat, _ sizetoFit: CGSize) -> UIImage {
        let rect = CGRect(origin: CGPoint(x: 0, y: 0), size: sizetoFit)

        UIGraphicsBeginImageContextWithOptions(rect.size, false, UIScreen.mainScreen().scale)
        CGContextAddPath(UIGraphicsGetCurrentContext(),
            UIBezierPath(roundedRect: rect, byRoundingCorners: UIRectCorner.AllCorners,
                cornerRadii: CGSize(width: radius, height: radius)).CGPath)
        CGContextClip(UIGraphicsGetCurrentContext())

        self.drawInRect(rect)
        CGContextDrawPath(UIGraphicsGetCurrentContext(), .FillStroke)
        let output = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();

        return output
    }
}
```

这种方式如果不做缓存，每次都会创建新的 UIImage，加重 CPU 的负担。

另外，还有一种使用贝塞尔曲线，利用CALayer层绘制指定圆角样式的mask遮盖View 的方式达到圆角效果的。不过这种方式也是操作 mask，会产生离屏渲染，但是效果会比直接设置圆角要好很多。

> 这样是把  GPU 的任务转给了 CPU 去完成。那么如果还是掉帧怎么办？
>
> 1. 直接让 UI 将图片切为圆角
> 2. 在原来的视图上添加一个四个角有颜色中间透明的图片,遮盖到原来图片上。这样是通过混合图层的方式实现。损耗的性能会好很多。
> 3. 将上面绘制圆角图片的过程放到子线程中去，绘制完成后回到主线程中。

针对上面第二点：添加一个四个角有颜色中间透明的图片。相关代码如下：

```objc
@implementation UIImage (XWAddForRoundedCorner)

/**提供一个在一个指定的size中绘制图片的便捷方法*/
+ (UIImage *)xw_imageWithSize:(CGSize)size drawBlock:(void (^)(CGContextRef context))drawBlock {
    if (!drawBlock) return nil;
    UIGraphicsBeginImageContextWithOptions(size, NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    if (!context) return nil;
    drawBlock(context);
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}

/**绘制方法的具体逻辑，遮罩图片的逻辑是绘制一个矩形，然后在绘制一个相应的圆角矩形，然后填充矩形和圆角矩形的中间部分为父视图的背景色*/
+ (UIImage *)xw_maskRoundCornerRadiusImageWithColor:(UIColor *)color cornerRadii:(CGSize)cornerRadii size:(CGSize)size corners:(UIRectCorner)corners borderColor:(UIColor *)borderColor borderWidth:(CGFloat)borderWidth{
    return [UIImage xw_imageWithSize:size drawBlock:^(CGContextRef  _Nonnull context) {
        CGContextSetLineWidth(context, 0);
        [color set];
        CGRect rect = CGRectMake(0, 0, size.width, size.height);
        //绘制一个矩形，这里发-0.3是为了防止边缘的锯齿，
        UIBezierPath *rectPath = [UIBezierPath bezierPathWithRect:CGRectInset(rect, -0.3, -0.3)];
        //绘制圆角矩形，这里的0.3是为了防止内边框的锯齿
        UIBezierPath *roundPath = [UIBezierPath bezierPathWithRoundedRect:CGRectInset(rect, 0.3, 0.3) byRoundingCorners:corners cornerRadii:cornerRadii];
        [rectPath appendPath:roundPath];
        CGContextAddPath(context, rectPath.CGPath);
        //注意要用EOFill方式进行填充而非Fill方式
        CGContextEOFillPath(context);
        //如下是绘制边框，原理依旧是绘制一个外边框然后根据边框宽度绘制一个内边框同样采取EOFill的方式进行填充即可
        if (!borderColor || !borderWidth) return;
        [borderColor set];
        UIBezierPath *borderOutterPath = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corners cornerRadii:cornerRadii];
        UIBezierPath *borderInnerPath = [UIBezierPath bezierPathWithRoundedRect:CGRectInset(rect, borderWidth, borderWidth) byRoundingCorners:corners cornerRadii:cornerRadii];
        [borderOutterPath appendPath:borderInnerPath];
        CGContextAddPath(context, borderOutterPath.CGPath);
        CGContextEOFillPath(context);
    }];
}
```



### 添加shadow的离屏渲染

我们直接设置 shadow 相关属性会产生离屏渲染：

```objc
let layer = view.layer
layer.shadowColor = UIColor.black.cgColor
layer.shadowOpacity = 0.3
layer.shadowRadius = radius
layer.shadowOffset = CGSize(width: 0, height: -3)
```

只要你提前告诉CoreAnimation你要渲染的View的形状Shape,就会减少离屏渲染计算。因此，我们需要加上设置 `shadowPath` 的一行：

```objc
let layer = view.layer
layer.shadowColor = UIColor.black.cgColor
layer.shadowOpacity = 0.3
layer.shadowRadius = radius
layer.shadowOffset = CGSize(width: 0, height: -3)
layer.shadowPath = [[UIBezierPathbezierPathWithRect：myView.bounds] CGPath];
```

## 参考文档 

[离屏渲染优化详解：实例示范+性能测试](https://www.jianshu.com/p/ca51c9d3575b)

