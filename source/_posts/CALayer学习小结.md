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

### 使用
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

## 创建图层
### 基本方法
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

## CAlayer属性
### position和anchorPoint
- position:设置CALayer在父层中的位置，这个位置要和锚点重合。
- anchorPoint:决定着CALayer身上的哪个点会在position属性所指的位置。以自己的左上角为原点(0, 0)，它的x、y取值范围都是0~1，**默认值为（0.5, 0.5）**

### 隐式动画
每个View内部都关联着一个Root Layer。所有非rootlayer都存在着隐式动画。隐式动画就是对于layer的部分属性进行修改时，默认会产生的一些动画。

常见的动画属性：
- bounds：用于设置CALayer的宽度和高度。修改这个属性会产生缩放动画
- backgroundColor：用于设置CALayer的背景色。修改这个属性会产生背景色的渐变动画
- position：用于设置CALayer的位置。修改这个属性会产生平移动画

## 自定义layer
### 第一种方式：新建layer类
想要在view中画东西，需要自定义view,创建一个类与之关联，让这个类继承自UIView，然后重写它的DrawRect：方法，然后在该方法中画图。

如果在layer上画东西，与上面的过程类似。
```objc
#import "YYMylayer.h"
@implementation YYMylayer
//重写该方法，在该方法内绘制图形
-(void)drawInContext:(CGContextRef)ctx
{
    //1.绘制图形
    //画一个圆
    CGContextAddEllipseInRect(ctx, CGRectMake(50, 50, 100, 100));
    //设置属性（颜色）
//    [[UIColor yellowColor]set];	不能这样设置
    CGContextSetRGBFillColor(ctx, 0, 0, 1, 1);
     //2.渲染
    CGContextFillPath(ctx);
}
@end
```
注意：
1. 默认为无色，不会显示。要想让绘制的图形显示出来，还需要设置图形的颜色。注意不能直接使用UI框架中的类
2. 在自定义layer中`drawInContext:`方法不会自己调用，只能自己通过`setNeedDisplay`方法调用，在view中画东西`DrawRect:`方法在view第一次显示的时候会自动调用。

在view中绘图：
```objc
#import "YYVIEW.h"
@implementation YYVIEW
- (void)drawRect:(CGRect)rect
{
    //1.获取上下文
    CGContextRef ctx=UIGraphicsGetCurrentContext();
    //2.绘制图形
    CGContextAddEllipseInRect(ctx, CGRectMake(50, 50, 100, 100));
    //设置属性（颜色）
    //    [[UIColor yellowColor]set];
    CGContextSetRGBFillColor(ctx, 0, 0, 1, 1);
    //3.渲染
    CGContextFillPath(ctx);
    //在执行渲染操作的时候，本质上它的内部相当于调用了下面的方法
    //[self.layer drawInContext:ctx];
}
```

### 第二种方式：实现delegate方法
设置viewcontroller为CALayer的delegate，然后让delegate实现drawLayer:inContext:方法，当CALayer需要绘图时，会调用delegate的drawLayer:inContext:方法进行绘图。
```objc
@implementation YYViewController
- (void)viewDidLoad{
    [super viewDidLoad];
    //1.创建自定义的layer
    CALayer *layer=[CALayer layer];
    //2.设置layer的属性
	...
    //设置代理
    layer.delegate=self;
    [layer setNeedsDisplay];
    //3.添加layer
    [self.view.layer addSublayer:layer];
}
-(void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx{
    //1.绘制图形
    //画一个圆
    CGContextAddEllipseInRect(ctx, CGRectMake(50, 50, 100, 100));
    //设置属性（颜色）
    //    [[UIColor yellowColor]set];
    CGContextSetRGBFillColor(ctx, 0, 0, 1, 1);   
    //2.渲染
    CGContextFillPath(ctx);
}
@end
```
注意：
1. 在设置代理的时候，它并不要求我们遵守协议，说明这个方法是nsobject中的，就不需要再额外的显示遵守协议了。
2. 不能再将某个UIView设置为CALayer的delegate，因为UIView对象已经是它内部根层的delegate，再次设置为其他层的delegate就会出问题。

### 补充
1. 无论采取哪种方法来自定义层，都必须调用CALayer的setNeedsDisplay方法才能正常绘图。
2. **当UIView需要显示时**，它内部的层会准备好一个CGContextRef(图形上下文)，然后**调用delegate(这里就是UIView)**的drawLayer:inContext:方法，并且传入已经准备好的CGContextRef对象。而UIView在drawLayer:inContext:方法中又会调用自己的drawRect:方法。平时在drawRect:中通过UIGraphicsGetCurrentContext()获取的就是由层传入的CGContextRef对象，在drawRect:中完成的所有绘图都会填入层的CGContextRef中，然后被拷贝至屏幕。

>Demo 详见 CALayer-transform
