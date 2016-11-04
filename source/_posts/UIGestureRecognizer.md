title: UIGestureRecognizer
date: 2016/9/5 14:07:12  
categories: iOS
tags: 
	- UIGesture
	
---

在iOS系统中，手势是进行用户交互的重要方式，通过`UIGestureRecognizer`类，我们可以轻松的创建出各种手势应用于app中。关于`UIGestureRecognizer`类，是对iOS中的事件传递机制面向应用的封装，将手势消息的传递抽象为了对象。有关消息传递的讨论，在上一篇中已经讨论过了。

<!--more-->

## 手势的抽象类——UIGestureRecognizer
`UIGestureRecognizer`将一些和手势操作相关的方法抽象了出来，但它本身并不实现什么手势，因此，在开发中，我们一般不会直接使用`UIGestureRecognizer`的对象，而是通过其子类进行实例化，iOS系统给我们提供了许多用于我们实例的子类，这些我们后面再说，我们先来看一下，`UIGestureRecognizer`中抽象出了哪些方法。

### 初始化方法
`UIGestureRecognizer`类为其子类准备好了一个统一的初始化方法，可以使用下面的方法进行统一的初始化：
```objc
- (instancetype)initWithTarget:(nullable id)target action:(nullable SEL)action;
```
如果我们使用`alloc-init`的方式，也是可以的，下面的方法可以为手势添加触发的`selector`：
```objc
- (void)addTarget:(id)target action:(SEL)action;
```
与之相对应的，我们也可以将一个`selector`从其手势对象上移除：
```objc
- (void)removeTarget:(nullable id)target action:(nullable SEL)action;
```
这里的`target`一般都是`self`,因为`selector`一般都是在`self`中定义。

因为`addTarget`方式的存在，iOS系统允许一个手势对象可以添加多个`selector`触发方法，并且触发的时候，所有添加的`selector`都会被执行，我们以点击手势示例如下:
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
     UITapGestureRecognizer * ges = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(click:)];
    [ges addTarget:self action:@selector(haha)];
     [self.view addGestureRecognizer:ges];
}
-(void)click:(UIGestureRecognizer *)ges{
    
    NSLog(@"第一个手势的触发方法");
    
}
-(void)haha{
    NSLog(@"haha");
}
```
点击屏幕后，两个方法都会触发。

### 手势状态
UIgestureRecognizer类中有如下一个属性，里面枚举了一些手势的当前状态:
```objc
@property(nonatomic,readonly) UIGestureRecognizerState state;
```
枚举值如下：
```objc
typedef NS_ENUM(NSInteger, UIGestureRecognizerState) {
    UIGestureRecognizerStatePossible,   // 默认的状态，这个时候的手势并没有具体的情形状态
    UIGestureRecognizerStateBegan,      // 手势开始被识别的状态
    UIGestureRecognizerStateChanged,    // 手势识别发生改变的状态
    UIGestureRecognizerStateEnded,      // 手势识别结束，将会执行触发的方法
    UIGestureRecognizerStateCancelled,  // 手势识别取消
    UIGestureRecognizerStateFailed,     // 识别失败，方法将不会被调用
    UIGestureRecognizerStateRecognized = UIGestureRecognizerStateEnded 
};
```
可以在手势的处理方法中，判断手势的状态，区分不同的处理方式。

### 常用属性和方法
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
可以通过设置`numberOfTapsRequired`来区分单击双击：
```objc
UITapGestureRecognizer *doubleTap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(DoubleTap:)];
doubleTap.numberOfTapsRequired = 2; //点击的次数 ＝2 双击
[_imageView addGestureRecognizer:doubleTap];//给对象添加一个手势监测；
```

另外还有三个属性用来控制触摸事件：
```objc
@property(nonatomic) BOOL cancelsTouchesInView;
@property(nonatomic) BOOL delaysTouchesBegan;
@property(nonatomic) BOOL delaysTouchesEnded;
```

**cancelsTouchesInView**
默认为YES,这种情况下当手势识别器识别到touch之后，会发送`touchesCancelled:withEvent:`消息,终止触摸事件的传递,以取消 `UIResponder`对touch的响应，这个时候只有手势识别器响应touch。
当设置成NO时，手势识别器识别到touch之后不会发送`touchesCancelled:withEvent:`消息，这个时候手势识别器和`UIResponder`均响应touch。
**注意：**发送`touchesCancelled:withEvent:`消息也是需要时间的。在每次手势操作的时候都会先触发1-2次touch操作，然后`touchesCancelled:withEvent:`消息才会生效，屏蔽touch操作。

**delaysTouchesBegan**
在前面`cancelsTouchesInView`属性为NO的基础上.
默认是NO，这种情况下当发生一个touch时，手势识别器先捕捉到到touch，然后发给`UIResponder`，两者各自做出响应。
如果设置为YES，手势识别器在识别的过程中（注意是识别过程），不会将touch发给`UIResponder`，即`UIResponder`不会有任何触摸事件。只有在识别失败之后才会将touch发给`UIResponder`，这种情况下`UIResponder`的响应会延迟约0.15ms。

**delaysTouchesEnded**
在前面`cancelsTouchesInView`属性为NO的基础上.
默认为YES。这种情况下发生一个touch时，在手势识别成功后,发送给`touchesCancelled:withEvent:`消息给`UIResponder`，手势识别失败时，会延迟大概0.15ms,期间没有接收到别的touch才会发送`touchesEnded:withEvent:`。
如果设置为NO，则不会延迟，即会立即发送`touchesEnded:withEvent:`以结束当前触摸。

### 手势间的互斥
有一点需要注意，同一个View上是可以添加多个手势对象的，默认这个手势是互斥的，一个手势触发了就会默认屏蔽其他相似的手势动作，例如：
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
     UITapGestureRecognizer * ges = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(click:)];;
    
    //view.backgroundColor = [UIColor redColor];
   
    //ges.delegate=self;
    [self.view addGestureRecognizer:ges];
    
    UITapGestureRecognizer * ges2 = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(click1:)];
//    ges2.delegate=self;
        [self.view addGestureRecognizer:ges2];
}


-(void)click:(UIGestureRecognizer *)ges{
    NSLog(@"第一个手势的触发方法");
}
-(void)click1:(UIGestureRecognizer *)ges1{
    NSLog(@"第二个手势的触发方法");   
}
```

我们添加的两个手势都是单机手势，会产生冲突，触发是很随机的，如果我们想设置一下当手势互斥时要优先触发的手势，可以使用如下的方法：
```objc
[ges requireGestureRecognizerToFail:ges2];
```
表示如果`ges2`匹配，那么不会执行`ges`。只有当`ges2`不匹配的时候，才会执行`ges`。

### UIGestureRecognizerDelegate
前面我们提到过关于手势对象的协议代理，通过代理的回调，我们可以进行自定义手势，也可以处理一些复杂的手势关系，其中方法如下：
```objc
//手指触摸屏幕后回调的方法，返回NO则不再进行手势识别，方法触发等
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch;
//开始进行手势识别时调用的方法，返回NO则结束，不再触发手势
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer;
//是否支持多时候触发，返回YES，则可以多个手势一起触发方法，返回NO则为互斥
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer;
//下面这个两个方法也是用来控制手势的互斥执行的
//这个方法返回YES，第一个手势和第二个互斥时，第一个会失效
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer NS_AVAILABLE_IOS(7_0);
//这个方法返回YES，第一个和第二个互斥时，第二个会失效
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldBeRequiredToFailByGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer NS_AVAILABLE_IOS(7_0);
```
最下面两个方法用起来还挺麻烦的，因为你不知道哪个是`gestureRecognizer`哪个是`otherGestureRecognizer`,因此，只能笼统的设置。


### 手势和触摸的区别
前一篇我们学习了`UITouch`,可以通过重写`UIResponder`的几个`touch`方法来处理触摸事件。
`UIGestureRecognizer`和触摸事件是两个独立的事,手势和`UITouch`本身关系不大。不过手势可以通过`cancelsTouchesInView`,`delaysTouchesBegan`,`delaysTouchesEnded`这三个属性来影响触摸事件。

## UIGestureRecognizer的子类
### 点击手势——UITapGestureRecognizer
点击手势十分简单，支持单击和多次点击，在我们手指触摸屏幕并抬起手指时会进行触发，其中有如下两个属性我们可以进行设置：
```objc
//设置点击次数，默认为单击
@property (nonatomic) NSUInteger  numberOfTapsRequired; 
//设置同时点击的手指数
@property (nonatomic) NSUInteger  numberOfTouchesRequired;
```

### 捏合手势——UIPinchGestureRecognizer
捏合手势是当我们双指捏合和扩张会触发动作的手势，我们可以设置的属性如下：
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

### 拖拽手势——UIPanGestureRecognzer
当我们点中视图进行慢速拖拽时会触发拖拽手势的方法。
```objc
//设置触发拖拽的最少触摸点，默认为1
@property (nonatomic)          NSUInteger minimumNumberOfTouches; 
//设置触发拖拽的最多触摸点
@property (nonatomic)          NSUInteger maximumNumberOfTouches;  
//获取当前位置
- (CGPoint)translationInView:(nullable UIView *)view;            
//设置当前位置
- (void)setTranslation:(CGPoint)translation inView:(nullable UIView *)view;
//设置拖拽速度
- (CGPoint)velocityInView:(nullable UIView *)view;
```
代码示例：
```objc
-(void)handlePan:(UIPanGestureRecognizer*)recognizer{
    NSLog(@"拖动操作");
    //处理拖动操作,拖动是基于imageview，如果经过旋转，拖动方向也是相对imageview上下左右移动，而不是屏幕对上下左右
    CGPoint translation = [recognizer translationInView:_imageView];
    recognizer.view.center = CGPointMake(recognizer.view.center.x + translation.x,
                                         recognizer.view.center.y + translation.y);
    [recognizer setTranslation:CGPointZero inView:_imageView];
}
```
必须使用`setTranslation`设置为`CGPointZero`,不知道为什么

### 滑动手势——UISwipeGestureRecognizer
滑动手势和拖拽手势的不同之处在于滑动手势更快，拖拽比较慢
```objc
//设置触发滑动手势的触摸点数
@property(nonatomic) NSUInteger                        numberOfTouchesRequired; 
//设置滑动方向
@property(nonatomic) UISwipeGestureRecognizerDirection direction;  
//枚举如下
typedef NS_OPTIONS(NSUInteger, UISwipeGestureRecognizerDirection) {
    UISwipeGestureRecognizerDirectionRight = 1 << 0,
    UISwipeGestureRecognizerDirectionLeft  = 1 << 1,
    UISwipeGestureRecognizerDirectionUp    = 1 << 2,
    UISwipeGestureRecognizerDirectionDown  = 1 << 3
};
```

### 旋转手势——UIRotationGestureRecognizer
进行旋转动作时触发手势方法
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

### 长按手势——UILongPressGestureRecognizer
进行长按的时候触发的手势方法
```objc
//设置触发前的点击次数
@property (nonatomic) NSUInteger numberOfTapsRequired;    
//设置触发的触摸点数
@property (nonatomic) NSUInteger numberOfTouchesRequired; 
//设置最短的长按时间
@property (nonatomic) CFTimeInterval minimumPressDuration; 
//设置在按触时时允许移动的最大距离 默认为10像素
@property (nonatomic) CGFloat allowableMovement;
```

### 手势组合的问题
这么多手势可以组合使用。但是使用的时候会产生如图所示的问题：
![手势组合](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/UIGestureRecognizer.gif?raw=true)
当图片正常大小时，拖动正常。当图片变小或者变大时，拖动距离变大以及变小。比如缩小图片后，相当于背景也缩小了，在手指滑动相同的距离下，相对来说移动的距离就变大了。

> Demo 详见UIGestureRecognizer