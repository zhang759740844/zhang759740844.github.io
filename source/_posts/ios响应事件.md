title: iOS手势与响应机制
date: 2016/9/3 10:07:12  
categories: iOS
tags:

​	- UIResponder

从点击屏幕到系统做出响应，经历了哪些过程？需要详细探究下ios的响应机制。本文参考自[史上最详细的iOS之事件的传递和响应机制
](http://www.jianshu.com/p/2e074db792ba)

<!--more-->

## UIResponder

### 点击事件响应流程

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uitouch1.png?raw=true)

#### 系统响应阶段

1. 手指触碰屏幕，事件交由 IOKit 处理
2. IOKit 将触摸事件封装为一个 Event 对象，通过 march port 传递给 SpringBoard 桌面进程。
3. SpringBoard 进程因接收到触摸事件，触发了其主线程 runloop 的 source1 事件源的回调。若前台无应用，则触发 SpringBoard 本身主线程 runloop 的 source0 事件源的回调，将事件交由桌面系统去消耗；若前台有应用，则将触摸事件通过IPC传递给前台APP进程。

#### APP响应阶段

1. APP进程的 mach port 接受到 SpringBoard 进程传递来的触摸事件，主线程的 runloop 被唤醒，触发了 source1 回调。
2. source1 回调又触发了一个 source0 回调，将接收到的 Event 对象封装成 UIEvent 对象，此时APP将正式开始对于触摸事件的响应。
3. source0 回调内部将触摸事件添加到 UIApplication 对象的事件队列中。事件出队后，UIApplication 开始通过不断 hit-testing 寻找最佳响应者。
4. 找到响应者后，事件便在响应链上传播。事实上，事件除了被响应者消耗，还能被手势识别器或是 target-action 模式捕捉并消耗掉。
5. 事件处理完毕后，runloop 进入休眠。等待再次被唤醒。

### 几个名词：响应者、触摸、事件

#### UIResponder

在iOS中不是任何对象都能处理事件，只有继承了`UIResponder`的对象才能接受并处理事件，我们称之为“响应者对象”。

以下都是继承自`UIResponder`的，所以都能接收并处理事件:
- UIApplication
- UIViewController
- UIView

那么为什么继承自UIResponder的类就能够接收并处理事件呢？因为UIResponder中提供了以下4个对象方法来处理触摸事件。
```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;
```

#### UITouch

- 一个手指一次触摸屏幕，就对应生成一个UITouch对象。多个手指同时触摸，生成多个UITouch对象。
- 每个UITouch对象记录了触摸的一些信息，包括触摸时间、位置、阶段、所处的视图、窗口等信息。

UITouch 提供了获取相对于视图的坐标的方法：

```objc
- (CGPoint)locationInView:(UIView *)view;
```

#### UIEvent

- 触摸的目的是生成触摸事件供响应者响应，一个触摸事件对应一个UIEvent对象
- UIEvent对象中包含了触发该事件的触摸对象的集合，可以通过**allTouches** 属性获取。

#### 例1：使用 UITouch 实现 UIView 的拖拽

通过 UIResponder 和 UITouch 的视图定位，可以实现拖拽 UI 的功能

```objc
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event{ 
    // 想让控件随着手指移动而移动,监听手指移动 
    // 获取UITouch对象 
    UITouch *touch = [touches anyObject]; 
    // 获取当前点的位置 
    CGPoint curP = [touch locationInView:self]; 
    // 获取上一个点的位置 
    CGPoint preP = [touch previousLocationInView:self]; 
    // 获取它们x轴的偏移量,每次都是相对上一次 
    CGFloat offsetX = curP.x - preP.x; 
    // 获取y轴的偏移量 
    CGFloat offsetY = curP.y - preP.y; 
    // 修改控件的形变或者frame,center,就可以控制控件的位置 
    // 形变也是相对上一次形变(平移) 
    // CGAffineTransformMakeTranslation:会把之前形变给清空,重新开始设置形变参数 
    // make:相对于最原始的位置形变 
    // CGAffineTransform t:相对这个t的形变的基础上再去形变 
    // 如果相对哪个形变再次形变,就传入它的形变 
    self.transform = CGAffineTransformTranslate(self.transform, offsetX, offsetY);
}
```

### 例2：配合 CAShapeLayer 实现一个画板

简单说就是记录下收拾的贝塞尔曲线，然后将这个曲线的 path 设置给 CAShapeLayer：

```objc
- (CAShapeLayer *)shapeLayer{
  if (_shapeLayer == nil) {
    _shapeLayer = [CAShapeLayer layer];
    _shapeLayer.strokeColor = [UIColor blackColor].CGColor;
    _shapeLayer.fillColor = [UIColor clearColor].CGColor;
    _shapeLayer.lineJoin = kCALineJoinRound;
    _shapeLayer.lineCap = kCALineCapRound;
    _shapeLayer.lineWidth = 10;
    [self.layer addSublayer:_shapeLayer];
   }
   return _shapeLayer;
 }

- (UIBezierPath *)path{
  if (_path == nil) {
    _path = [UIBezierPath bezierPath];
    _path.lineWidth = 10;
  }
  return _path;
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
  UITouch *touch = [touches anyObject];
  CGPoint point = [touch locationInView:self];
  [self.path addLineToPoint:point];
  self.shapeLayer.path = self.path.CGPath;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
  UITouch *touch = [touches anyObject];
  CGPoint point = [touch locationInView:self];
  [self.path moveToPoint:point];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
  self.shapeLayer = nil;
  self.path = nil;
}
```

### 向下寻找响应者的过程

#### 自上而下的寻找过程

1. UIApplication首先将事件传递给 UIWindow，多个窗口优先最上层的窗口。
2. 调用 `hitTest` 方法，询问是否可以响应。视图若不能响应，则将事件传递给上一个同级子视图；若能响应，则从后往前询问当前视图的子视图。
3. 重复步骤2。视图若没有能响应的子视图了，则自身就是最合适的响应者。

#### 无法响应的情况

- 不允许交互：`userInteractionEnabled = NO`
- 隐藏：如果把父控件隐藏，那么子控件也会隐藏，隐藏的控件不能接受事件
- 透明度：如果设置一个控件的透明度<0.01，会直接影响子控件的透明度。alpha：0.0~0.01为透明。

**注意**:

- 默认UIImageView不能接受触摸事件，因为不允许交互，即`userInteractionEnabled = NO`，所以如果希望UIImageView可以交互，需要`userInteractionEnabled = YES`。

#### 判断是否可以响应的逻辑

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    //3种状态无法响应事件
     if (self.userInteractionEnabled == NO || self.hidden == YES ||  self.alpha <= 0.01) return nil; 
    //触摸点若不在当前视图上则无法响应事件
    if ([self pointInside:point withEvent:event] == NO) return nil; 
    //从后往前遍历子视图数组 
    int count = (int)self.subviews.count; 
    for (int i = count - 1; i >= 0; i--) { 
        // 获取子视图
        UIView *childView = self.subviews[i]; 
        // 坐标系的转换,把触摸点在当前视图上坐标转换为在子视图上的坐标
        CGPoint childP = [self convertPoint:point toView:childView]; 
        //询问子视图层级中的最佳响应视图
        UIView *fitView = [childView hitTest:childP withEvent:event]; 
        if (fitView) {
            //如果子视图中有更合适的就返回
            return fitView; 
        }
    } 
    //没有在子视图中找到更合适的响应视图，那么自身就是最合适的
    return self;
}
```

`pointInside:withEvent:` 这个方法，用于判断触摸点是否在自身坐标范围内

#### 例1：超出父视图区域的点击响应

开发中可能会遇到一种情况：Tabbar 中的一个按钮超出了 tabbar 显示的区域。默认情况下，点击超出部分是无法响应点击事件的。

因为触摸点不在TabBar的坐标范围内，因此TabBar无法响应该触摸事件，`hitTest:withEvent:` 直接返回了nil。整个过程，事件根本没有传递到圆形按钮。所以我们需要做的是扩大 tabbar 的点击范围。

我们需要重写 tabbar 的 `pointInside:withEvent:` 方法，先把位置转换到按钮上判断一下是否点击了按钮：

```objc
//TabBar
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    //将触摸点坐标转换到在CircleButton上的坐标
    CGPoint pointTemp = [self convertPoint:point toView:_CircleButton];
    //若触摸点在CricleButton上则返回YES
    if ([_CircleButton pointInside:pointTemp withEvent:event]) {
        return YES;
    }
    //否则返回默认的操作
    return [super pointInside:point withEvent:event];
}
```

#### 例2：按钮扩大响应区域

和上面类似，还是重写 `pointInside:withEvent:` 方法：

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
  	// 将一个 100*100 的按钮扩大为 300*300 的按钮
  	CGRect rect = CGRectMake(-100, -100, 300, 300);
		return CGRectContainsPoint(rect, point) ? YES : NO;
}
```

### 事件响应的向上传递

在找到最佳响应者之后，UIApplication将事件通过 `sendEvent:` 传递给事件所属的window，window同样通过 `sendEvent:` 再将事件传递给最佳响应者：

```objc
UIApplication ——> UIWindow ——> hit-tested view
```

响应者接收到时间之后，会回调 `UIResponder` 中的 `touchesBegan:withEvent:` 方法。响应者对于接收到的事件有3种操作：

- 不拦截，默认操作事件会自动沿着默认的响应链往下传递
- 拦截，不再往下分发事件。重写 `touchesBegan:withEvent:` 进行事件处理，不调用父类的 `touchesBegan:withEvent:` 
- 拦截，继续往下分发事件。重写 `touchesBegan:withEvent:` 进行事件处理，同时调用父类的 `touchesBegan:withEvent:` 将事件往下传递

每一个响应者对象（UIResponder对象）都有一个 `nextResponder` 方法，用于获取响应链中当前对象的下一个响应者。调用父类的 `touchesBegan:withEvent:`  会默认调用 `nextResponder` 的 `touchesBegan:withEvent:`。因此，就把响应事件向上传递了。

> 我们可以通过响应链 `nextResponder` 找到下一级响应者，直到找到 UIViewController 的子类

## UIGestureRecognizer

### 手势简介

`UIGestureRecognizer` 是手势的基类，可以创建其派生类实例来满足不同需求。

#### 初始化方法

`UIGestureRecognizer`类为其子类准备好了一个统一的初始化方法：

```objc
- (instancetype)initWithTarget:(nullable id)target action:(nullable SEL)action;
```

#### 手势状态

UIgestureRecognizer类中有如下一个属性，里面枚举了一些手势的当前状态:

```objc
@property(nonatomic,readonly) UIGestureRecognizerState state;
```

枚举值如下：

```objc
typedef NS_ENUM(NSInteger, UIGestureRecognizerState) {
    UIGestureRecognizerStatePossible,   // 默认的状态，这个时候的手势并没有具体的情形状态
    UIGestureRecognizerStateBegan,      // 手势开始被识别的状态
    UIGestureRecognizerStateChanged,    // 手势识别发生改变的状态(手势正在移动的状态)
    UIGestureRecognizerStateEnded,      // 手势识别结束，将会执行触发的方法
    UIGestureRecognizerStateCancelled,  // 手势识别取消
    UIGestureRecognizerStateFailed,     // 识别失败，方法将不会被调用
    UIGestureRecognizerStateRecognized
};
```

可以在手势的处理方法中，判断手势的状态，区分不同的处理方式。

#### 常用属性和方法

```objc
//设置代理，具体的协议后面会说
@property(nullable,nonatomic,weak) id <UIGestureRecognizerDelegate> delegate; 
//设置手势是否有效
@property(nonatomic, getter=isEnabled) BOOL enabled;
//获取手势所在的view
@property(nullable, nonatomic,readonly) UIView *view; 
//获取触发触摸的点
- (CGPoint)locationInView:(nullable UIView*)view; 
//设置触摸点数
- (NSUInteger)numberOfTouches; 
//获取某一个触摸点的触摸位置
- (CGPoint)locationOfTouch:(NSUInteger)touchIndex inView:(nullable UIView*)view;
```

其中，`UITouch`也有一个方法是`locationInView:`可以获取触摸点在view中的位置。

#### 代理方法 UIGestureRecognizerDelegate

```objc
//手指触摸屏幕后回调的方法，返回NO则不再进行手势识别，方法触发等
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch;
//开始进行手势识别时调用的方法，返回NO则结束，不再触发手势
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer;
//是否支持多时候触发，返回YES，则可以多个手势一起触发方法，返回NO则为互斥
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer;
//下面这个两个方法也是用来控制手势的互斥执行的
//这个方法返回YES，第一个手势和第二个互斥时，第一个会失效
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
//这个方法返回YES，第一个和第二个互斥时，第二个会失效
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldBeRequiredToFailByGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
```

#### 手势互斥

同一个View上是可以添加多个手势对象的，默认这个手势是互斥的，并且触发是随机的，一个手势触发了就会默认屏蔽其他相似的手势动作。

如果我们想设置一下当手势互斥时要优先触发的手势，可以使用如下的方法：

```objc
[ges requireGestureRecognizerToFail:ges2];
```

表示如果`ges2`匹配，那么不会执行`ges`。只有当`ges2`不匹配的时候，才会执行`ges`。这个方法还适用于识别双击手势时屏蔽单击手势。只有确定不是双击手势后再识别为单击手势

### UIGestureRecoginzer 子类

#### 点击手势——UITapGestureRecognizer

```objc
//设置点击次数，默认为单击
@property (nonatomic) NSUInteger  numberOfTapsRequired; 
//设置同时点击的手指数
@property (nonatomic) NSUInteger  numberOfTouchesRequired;
```

#### 捏合手势——UIPinchGestureRecognizer

```objc
//设置缩放比例
@property (nonatomic)          CGFloat scale; 
//设置捏合速度
@property (nonatomic,readonly) CGFloat velocity;
```

在设置完缩放后，一定要把`recognizer.scale`设置为1

```objc
- (void)handlePinch:(UIPinchGestureRecognizer*)recognizer{
    NSLog(@"缩放操作");//处理缩放操作
    //对imageview缩放
    _imageView.transform = CGAffineTransformScale(_imageView.transform, recognizer.scale, recognizer.scale);
    recognizer.scale = 1;
}
```

#### 拖拽手势——UIPanGestureRecognzer

```objc
//设置触发拖拽的最少触摸点，默认为1
@property (nonatomic)          NSUInteger minimumNumberOfTouches; 
//设置触发拖拽的最多触摸点
@property (nonatomic)          NSUInteger maximumNumberOfTouches;  
//获取手势的当前位置
- (CGPoint)translationInView:(nullable UIView *)view;            
//设置手势的当前位置
- (void)setTranslation:(CGPoint)translation inView:(nullable UIView *)view;
```

拖动过程中处理方法会被多次调用，但是在一次拖拽结前，`translationInView:` 方法参照的点都是最开始按下的点。这就导致增量的拖动，越拖越快。所以我们必须使用`setTranslation`设置为`CGPointZero`，就能将手指的当前位置设置为拖移手势的起始位置：

```objc
-(void)handlePan:(UIPanGestureRecognizer*)recognizer{
    NSLog(@"拖动操作");
    //处理拖动操作,拖动是基于imageview，如果经过旋转，拖动方向也是相对imageview上下左右移动，而不是屏幕对上下左右
    // 拖动过程中可以判断是否为 UIGestureRecognizerStateChanged
    if (recognizer.state == UIGestureRecognizerStateChanged){
          CGPoint translation = [recognizer translationInView:_imageView];
    	  recognizer.view.center = CGPointMake(recognizer.view.center.x + translation.x, recognizer.view.center.y + translation.y);
          [recognizer setTranslation:CGPointZero inView:_imageView];
    }
}
```

#### 滑动手势——UISwipeGestureRecognizer

滑动手势和拖拽手势的不同之处在于滑动手势更快，拖拽比较慢

```objc
//设置触发滑动手势的触摸点数
@property(nonatomic) NSUInteger                        numberOfTouchesRequired; 
//设置滑动方向
@property(nonatomic) UISwipeGestureRecognizerDirection direction;  
```

#### 旋转手势——UIRotationGestureRecognizer

```objc
//设置旋转角度
@property (nonatomic)          CGFloat rotation;
//设置旋转速度 
@property (nonatomic,readonly) CGFloat velocity;
```

在设置完旋转后，`recognizer.rotation`一定要清零.

```objc
- (void)handleRotate:(UIRotationGestureRecognizer*) recognizer{
    NSLog(@"旋转操作");//处理旋转操作
    //对imageview旋转
    _imageView.transform = CGAffineTransformRotate(_imageView.transform, recognizer.rotation);
    recognizer.rotation = 0;    //一定要清零
}
```

#### 长按手势——UILongPressGestureRecognizer

```objc
//设置触发前的点击次数
@property (nonatomic) NSUInteger numberOfTapsRequired;    
//设置触发的触摸点数
@property (nonatomic) NSUInteger numberOfTouchesRequired; 
//设置最短的长按时间
@property (nonatomic) CFTimeInterval minimumPressDuration; 
//设置在按触时时允许移动的最大距离 默认为10像素
@property (nonatomic) CGFloat allowableMovemen
```

### 手势与响应链

#### 结论

event 绑定的 touch 对象上维护了一个手势识别器数组。在响应链通过 hit-test 寻找最佳响应视图的时候，会收集响应链上每一个视图上施加的手势：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uitouch2.png?raw=true)

Window先将事件传递给这些手势识别器，再传给hit-tested view。一旦有手势识别器成功识别了手势，Application就会取消hit-tested view对事件的响应。因此可以理解为**手势识别器比UIResponder具有更高的事件响应优先级**。

> 但是，识别手势是需要时间的，所以具体的表现就是当有点击事件的时候，会先触发 `touchBegan:withEvent:` 方法，然后当手势识别到的时候，就会触发 `touchCancelled:withEvent:`

比如在视图上添加一个 UIPanGestureRecognizer 手势。打印日志可以看到，会先触发 UIResponder 的回调，直到 UIPanGestureRecognizer 识别成功：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uitouch3.png?raw=true)

#### 手势识别器的两个属性

```objc
@property(nonatomic) BOOL cancelsTouchesInView;
@property(nonatomic) BOOL delaysTouchesBegan;
```

##### cancelsTouchesInView

默认为YES。表示当手势识别器成功识别了手势之后，会通知Application取消响应链对事件的响应，并不再传递事件给hit-test view。若设置成NO，表示手势识别成功后不取消响应链对事件的响应，事件依旧会传递给hit-test view。

如果设置 `pan.cancelsTouchesInView = NO`，那么上面的 UIPanGestureRecognizer 的日志会变为：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uitouch4.png?raw=true)

##### delaysTouchesBegan

默认为NO。默认情况下手势识别器在识别手势期间，当触摸状态发生改变时，Application都会将事件传递给手势识别器和hit-tested view；若设置成YES，则表示手势识别器在识别手势期间，截断事件，即不会将事件发送给hit-tested view。

如果设置 `pan.delaysTouchesBegan = NO`，那么上面的 UIPanGestureRecognizer 的日志会变为：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uitouch5.png?raw=true)

### UITableView 的点击和手势的冲突

UIScrollView 的滑动其实是因为系统加了一个 UIPanGesture的缘故。UITableView 点击 Cell 其实是调用了 `touchBegan:withEvent:` 方法。

因此，当有这样一个需求：cell 支持左滑，并且当一个 cell 左滑固定的时候，点击 UITableView 任意位置会关闭 cell。这种时候，需要给 UITableView 添加一个 UITapGesture。而这个 UITapGesture 就会导致 UITableView 本身 Cell 点击的不响应。

因此，我们需要在通常情况下设置这个 `tapGesture.enabled = NO;`。只有在 Cell 展开的情况下，设置 `tapGesture.enabled = YES`

### 手势与UIControl

UIControl 继承于 UIResponder。像 UIButton 就是继承于 UIControl

#### 结论

系统派生于 UIControl 的类像 UIButton 之类，处理事件的优先级比**父视图上的** UIGestureRecognizer高。UIControl会阻止父视图上的手势识别器行为。

> 注意，自己继承于 UIControl 实现的类不存在优先级比手势高的情况。
>
> 另外，UIControl 的优先级是比父视图上的手势高，如果当前视图也有手势，那么 UIControl 无法阻止手势响应。

## 一些技巧

### 几个坐标转换的方法

#### UITouch

```objc
// 返回值表示触摸在view上的位置
// 这里返回的位置是针对view的坐标系的（以view的左上角为原点(0, 0)）
// 调用时传入的view参数为nil的话，返回的是触摸点在UIWindow的位置
- (CGPoint)locationInView:(UIView *)view; 
// 该方法记录了前一个触摸点的位置
- (CGPoint)previousLocationInView:(UIView *)view; 
```

#### UIGestureRecognizer

```objc
// 返回坐标点,第一个参数为tauch数组的索引
CGPoint point= [pan locationOfTouch:0 inView:self.view];
// 手指移动了多少，上面是相对于坐标原点，这个是相对于拖拽的起始点，用于 拖拽 View 时候修改 View 位置的
// 配合 [pan setTranslation:CGPointZero inView: someView] 使用，每次拖拽方法回调的时候，都要将上次的拖拽位移清零
CGPoint point = [pan translationInView:self.view];
```

#### UIView

```objc
// UIView 转换到另一个 UIView 坐标系，在 hittest 中判断子控件是否是 responder 的时候会使用
- (CGPoint)convertPoint:(CGPoint)point toView:(UIView *)view;
```



[iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

