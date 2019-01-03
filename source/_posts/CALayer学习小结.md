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
UIView之所以能显示在屏幕上，因为内部的layer。创建UIView的时候自动创建CALayer对象。
当UIView需要显示到屏幕上时，会调用drawRect:方法进行绘图，并且会将所有内容绘制在自己的图层上，绘图完毕后，系统会将图层拷贝到屏幕上，于是就完成了UIView的显示。
UIView本身不具备显示的功能，拥有显示功能的是它内部的图层。

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

### position和anchorPoint

- position:设置CALayer在父层中的位置，这个位置会和锚点重合。
- anchorPoint:决定着CALayer身上的哪个点会在position属性所指的位置。以自己的左上角为原点(0, 0)，它的x、y取值范围都是0~1，**默认值为（0.5, 0.5）**。layer 的缩放和旋转，都是以 anchorPoint 为原点的。

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
> 还有一种更好的办法，直接用 view 的 transform 属性。

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



## Mask属性

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


