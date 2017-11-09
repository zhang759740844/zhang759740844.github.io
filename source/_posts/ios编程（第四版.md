title: ios编程(第四版) 学习笔记
date: 2016/7/31 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

本文是对《ios编程(第四版)》一书学习后摘录的笔记，属于ios开发基础。

<!--more-->




## 视图与视图层次结构

### 视图层次结构

视图层次结构中的每个视图分别绘制自己。视图会将自己绘制到图层(layer)上，每个 **UIView** 对象都有一个 layer 属性，指向一个 **CALayer** 类的对象。

> layer 只负责显示，layer 可以不和 View 重叠的放在任何位置，但是关于视图的响应还是要靠 UIView，即看 View 的位置。

![视图首先分别绘制自己，然后组合起来绘制到屏幕上](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_视图.jpg?raw=true)



### 创建 UIView 子类

每个 **UIView** 对象都有一个 superview 属性。将一个视图作为子视图加入另一个视图时，会自动创建相应的反向关联。



### 在 drawRect: 方法中自定义绘图

视图根据 **drawRect:** 方法将自己绘制到图层上。每个 **UIView** 的子类可以覆盖 **drawRect:**，完成自定义的绘图任务。例如 **UIButton** 的 **drawRect:** 方法默认会在 frame 表示的矩形区域中心画一行浅蓝色的文字。

可以使用 **UIBezeierPath** 绘制直线或者曲线来组成各种形状，例如画一个同心圆的一个略伪的代码：

```objc
- (void)drawRect:(CGRect)rect{
  UIBezierPath *path = [[UIBezierPath alloc] init];
  
  // 画同心圆
  for (float currentRadius = maxRadius; currentRadius >0; currentRadius -= 20){
    // 每一次一个 path 的 stroke 都相当于是一笔画，绘制完一个圆后必须要抬笔让起始点移动到另一个地方
    [path moveToPoint:CGPointMake(center.x + currentRadius, center.y)];
    // 画弧
    [path addArcWithCenter:center radius:currentRadius startAngle:0.0 endAngle:M_PI*2 clockwise:YES];
  }
  
  // 设置线条宽度为10
  path.lineWidth = 10;
  
  // 设置画笔颜色为浅灰色
  [[UIColor lightGrayColor] setStroke];
  
  // 绘制路径
  [path stroke];
}
```



### 深入学习: Core Graphics

 Core Graphics 中最重要的对象是图形上下文，图形上下文是 CGContextRef 的对象，负责存储回话状态(例如画笔颜色和线条粗细)和挥之内容所处的内存空间。

参与绘图操作的类都定义了改变绘图状态和执行绘图操作的方法，这些方法其实调用了对应的 Core Graphics 函数。有时候我们不得不用到其中的一些属性，但是一般情况下，我们没有必要直接使用 Core Graphics。



## 视图:重绘与 UIScrollView

### 运行循环和重绘视图

iOS 应用启动会开始一个运行循环。运行循环的工作是处理事件，例如触摸。运行循环会为相应的事件找到合适的处理方法。当这些方法执行完毕后，控制权会再次回到运行循环。

当应用将控制权交回给运行循环时，运行循环首先会检查是否有等待重绘的视图(即当前循环收到过 **setNeedDisplay** 消息的视图)，然后向所有等待重绘的视图发送 **drawRect:**

在事件处理周期中，视图的属性可能会多次改变，如果经常重绘自己，会减慢界面的响应速度。iOS 会在运行循环的最后阶段集中处理所有需要重绘的视图，尤其是对于属性多次发生改变的视图，在每次时间处理周期中只重绘一次。

为了标记视图要重绘，必须向其发送 **setNeedsDisplay** 消息。系统中的视图对象会在显示的内容发生改变时向自身发送 **setNeedsDisplay** 消息，例如 **UILabel** 对象收到 **setText:** 消息后，会自动标记为需要重绘。而自定义的 **UIView** 子类，必须手动向其发送 **setNeedsDisplay** 消息。

![视图在运行循环中重绘自己](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_运行循环.jpg?raw=true)

```objc
- (void)setCircleColor:(UIColor *)circleColor{
  circleColor = circleColor;
  [self setNeedsDisplay];	// 重绘
}
```



### 使用 UIScrollView

另一篇专门讲 **UIScrollView** 的文章已经说了，**UIScrollView** 有两个 size，一个控制显示的框框的大小，就是指 **frame**，一个是实际内部图像的大小，就是其中的 **contentSize** 属性：

```objc
// 外部大小设置为屏幕大小
UIScrollView *scrollView = [[UIScrollView alloc] initWithFrame:screenRect];
// 添加一个 ContentView，在这个 View 上添加更多要显示的图形
UIView *contentView = [[UIView alloc] initWithFrame:bigRect];
[scrollView addSubview:contentView];
// 告诉 UIScrollView 取景范围多大
scrollView.contentSize = bigRect.size;

// 可以在 contentView 中添加各种 subview
[contentView addSubview:xx1View];
[contentView addSubview:xx2View];
```



**UIScrollView** 中有一个分页的属性 **pagingEnabled**，如果将其设置为 YES，会一页一页地显示视图，其实现原理是： **UIScrollView** 对象会根据其 bounds 的尺寸，将 contentSize 分割为尺寸相同的多个区域。拖动结束后， **UIScrollView** 会自动滚动并值显示其中一个区域。



## 视图控制器

### 创建 UIViewController子类

**UIViewController**有一个重要属性 view，它是视图层次结构的根视图，所有视图都添加在这个 view 上:

```objc 
@property (nonatomic, strong) UIView *view;
```

视图控制器可以通过两种方式创建视图层次结构:

- 代码方式:覆盖 **UIViewController** 中的 **loadView** 
- NIB 文件方式:不覆盖 **UIViewController** 中的 **loadView** 

**loadView** 方法会在 ViewController 的 view 被请求并且**当前view为nil时**调用这个方法。所以你调用 `[[UIViewController alloc] init]` 并且将这个 ViewController 显示出来的时候，会自动触发 **loadView** 方法。

系统的 **loadView** 方法的默认实现是：寻找有关的 nib 文件的信息，根据这个信息来加载 nib 文件。如果没有相关nib文件的信息，默认创建一个空白的 UIView 对象，然后把对象成赋值给 ViewController 的主 view。

这个nib文件的信息是 **initWithNibName:Bundle:** 方法提供的，你可以在创建 VC 的时候直接手动调用：`[[UIViewController alloc] initWithNibName:@"xxxx" Bundle:[UIBundle mainBundle]]`。如果是使用 init 方法初始化的，init 方法会默认调用该方法，其中的 nibName 默认是同名文件。

> 通过 Nib 文件方式加载视图就是不要重写 loadView 方法

自定义重载 **loadView** 的时候注意不需要调用上面所说的系统实现方法 `[super loadView];` 因为你想自定义 view 的话，就需要自己给 view 属性赋值。

```objc
- (void)loadView{
  // 创建一个view
  UIView *view = [[UIView alloc] init];
  // 将 view 赋给控制器的 view 属性
  self.view = view;
  // 添加其他的view
  ...
}
```

> 重写 loadView 来实现代码方式自定义视图

无论 XIB 还是代码创建都会调用 **viewDidLoad** 方法。**如果你需要对 view 及添加在其上的视图做一些其他的定制操作，在 viewDidLoad 里面去做。**



### 另一个视图控制器

通过 **initWithNibName:Bundle:** 方法可以指定加载的 NIB 文件，bundle 用于指定所在的程序包，默认是主程序包 `[NSBundle mainBundle];`，对应于文件系统中的根目录，包含代码文件和资源文件。

在加载前，我们要确保 nib 对象和 ViewController 已关联起来(**尤其是 view 属性要关联起来**，不关联就会抛出异常)，需要使用 File's Owner 对象。File's Owner 对象是一个占位符，当某个视图控制器将 Xib 文件加载为 NIB 文件时，首先会创建 XIB 文件中所有视图对象，然后会将自己填入相应的 File's Owner 空洞，并建立之前在 Interface Builder 中设置的关联。

![NIB文件加载过程的时间轴](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_fileowner.jpg?raw=true)

关联 view 属性，首先选中 Files's Owner 然后打开检视面板，将 Class 设置为目标 ViewController 的 Class：

![针对 File's Owner 的标识检视面板](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_ViewControllerClass.jpg?raw=true)

设置 File's Owner 是在 **initWithNibName:bundle:** 方法里设置的，那么上图的设置意义是什么呢？只有正确的设置了 Class，下面关联插座的时候才能将 ViewController 中的插座变量展示出来，提供给使用者连线：

![关联 view 插座变量](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_Outlet.jpg?raw=true)

比如**在 NavigationController 中的一堆 ViewController，在内存偏少时，会释放其视图并在显示的时候再创建出来。所以 ViewController 中的插座变量一般声明为 weak，防止 ViewController 强引用 Xib，导致系统不能释放 Xib。**

### **插播一：File‘s Owner 和 View 的 custom class 的区别**

#### 用于 ViewController 上的区别

除了 File's Owner 可以设置 custom class，xib 中的 view 也可以设置 custom class。但是**前者可以是任意对象即 NSObject 而后者只能是 UIView 的子类**。我们在对 ViewController 设置 xib 的时候只可以设置 File's Owner 的 custom class，并通过 **initWithNibName:bundle:** 初始化，而不能设置 xib 中 view 的 custom class。

因为 UIViewController 继承于 NSObject，本质上是一个控制器，而不是一个 View。所以这个 xib 所代表的 View 是 custom class 也就是 VC 的一部分。从连线上就可以作证，因为如果创建 VC 的时候连同创建了 xib 的话，Xcode 会默认将 VC 的 View 属性和 xib 连接。

而 View 则表示 custom class 就是这个 view。所以不能将其用在 VC 上。

#### 用于 View 上的区别

那对于一个自定义的 View，我们是该设置哪个的 custom class 呢？答案是：两者都可以实现，但是使用场景和方式都略有区别。

对于自定义 view，两者都可以使用下面的方法表示。前者需要将 `owner` 设置为现在的 owner，一般是 self；后者将其置为 nil 即可：

```objc
NSArray *nibViews = [[NSBundle mainBundle] loadNibNamed:@"NibName" owner:owner options:nil];
```



如果是设置 view 的 custom class。例如一个 TestView 视图，直接就可以将这个 View 赋给了 TestView 对象。这里设置 `owner` 为 nil：

```objc
// 在某个 ViewController 中添加 TestView 视图
NSArray* nibView = [[NSBundle mainBundle] loadNibNamed:@"TestView" owner:nil options:nil];
TestView *testView = [nibView firstObject];
[self.view addSubview:testView];
```

设置 Files's Owner 的 custom class 的方式则需要通过 `addSubView:` 方法手动添加到其 owner 类中：

```objc
// 自定义的 TestView 的某个实例化方法
- (instancetype)initFromNibWithFrame:(CGRect)frame{
    UIView *nibView = [[[NSBundle mainBundle] loadNibNamed:@"TestView" owner:self options:nil] firstObject];
    nibView.frame = frame;
    self = [super initWithFrame:nibView.frame];
    [self addSubview:nibView];
 	// 这个自定义的方法用来自定义设置 View
    [self nib_viewDidLoad];
    return self;
}
```

一般我们如果自定义 View 都是直接采用设置 View 的 custom class 的方式，比如 TableView 中加载头视图，设置 TableViewCell 等：

```objc
// 设置头视图
UIView *headerView = [[[NSBundle mainBundle] loadNibNamed:@"HotelReviewsHeaderView" owner:nil options:nil]lastObject];

// 设置 Cell
[self.tableView registerNib:[UINib nibWithNibName:@"MineUserInfoCell" bundle:nil]  forCellReuseIdentifier:@"MineUserInfoCellIdentifier"];
```

> 设置 File‘s owner 就表示这个 xib 是我 Custom class 的马仔，所以要 `addSubview` 的方式添加进去。设置 View 的custom class 表示这个 xib 就是我自己呀。

参考：[通过 XIB 加载 UIView](http://honglu.me/2016/02/22/通过XIB加载UIView/)



### 插播二：在 xib 中使用自定义的view

上面演示了如何用代码的方式对设置了 File's owner 或 View 的 custom view 的自定义视图进行实例化。那么一直有一个疑惑，如何在 xib 中直接添加自定义view 的 xib？

在 ViewController 的 xib 中，我们拖入一个 UIView 的控件，然后将这个 UIView 的 Custom Class 设置为自定义的 view 的类名。此刻你觉得大功告成了么？非也，如果运行就会发现，刚刚拖入并设置了 Custom Class 的控件并没有显示出来，只是一片空白。那么我们要如何修改呢？

> 其实为什么是一片空白很好理解，因为你拖的这个 UIView 就是空白的，Xcode 就认为你将类和这个空白的 UIView 关联了起来。
>
> 虽然你还将另外的一个 xib 和这个类关联起来了，但是你显示的并不是那个 xib。其实就相当于一个类关联了多个 xib 的视图。

当指定了类的那个 UIView 随着 **ViewController 的 xib** 被加载的时候，会调用 `initWithCoder:` 以及 `awakeFromeNib` 方法（只要是从文件(即 xib)中加载的，就都会调用其类的该方法。其实就相当于在加载 VC 的 xib 的时候，递归的调用了其内部所有 View 的 `initWithCoder` 方法）。我们可以重写 `initWithCoder:` 方法将我们要展示的 UIView 添加上去。

> `initWithCoder` 是文件方式创建 UIView 的时候就会调用的，调用 `[super initWithCoder:aDecoder]` 就完成了 UIView 的创建。你可以添加自己的代码，在 UIView 中添加视图。
>
> `initWithFrame` 就是以代码方式创建 UIView 的了。
>
> `awakeFromNib` 是在 `initWithCoder` 成功从文件创建好 UIView 之后调用的。

这里就不能像前面一样可以设置 File's Owner 和 View 的 custom class 都可以加载自定义视图对象了。这里只能设置 File's Owner 的 custom class，然后在 `initWithCoder:` 方法中添加 subView。最后注意由于是添加的 subView，那么需要设置 frame。

```objc
- (instancetype)initWithCoder:(NSCoder *)aDecoder
{
    self = [super initWithCoder:aDecoder];
    if (self) {
        NSString *className = NSStringFromClass([self class]);
        UIView *view = [[[NSBundle mainBundle] loadNibNamed:className owner:self options:nil] firstObject];
        view.frame = self.bounds;
        [self addSubview:view];
        return self;
    }
    return nil;
}
```

通过设置 View 的 custom class 将会产生无线递归，最终导致栈溢出崩溃。为什么？因为 **如果你在 xib 中将 View 的 custom class 设置为自定义view 的类**，那么你在 `initWithCoder` 中调用 `UIView *view = [[[NSBundle mainBundle] loadNibNamed:className owner:self options:nil] firstObject];` 的时候会触发 `initWithCoder:`。所以这个过程相当于不停的在加载自己。只有当你将其设置为 File's Owner 的时候，在 `initWithCoder` 中加载视图，就是加载子视图，就相当于加载自己的儿子。

[参考：Nested Xib Views - 使用XIB实现嵌套自定义视图](http://blog.wtlucky.com/blog/2014/08/10/nested-xib-views/)



### UITabBarController

**UITabBarController** 的 **view** 属性指向一个包含两个子视图的 **UIView** 对象，分别是标签栏和当前选中的视图控制器的视图。其 **viewControllers** 属性保存一组视图控制器，每一个视图控制器对应一个 tabItem：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions{
  UIImage *mainImage = [UIImage imageNamed:@"tab_home"];
  UIImage *mainImageSel = [UIImage imageNamed:@"tab_home_active"];
  //设置图片的渲染模式
  mainImage = [mainImage imageWithRenderingMode:UIImageRenderingModeAlwaysOriginal];
  mainImageSel = [mainImageSel imageWithRenderingMode:UIImageRenderingModeAlwaysOriginal];
  //设置tabBarItem
  UITabBarItem *mainTabBarItem = [[UITabBarItem alloc] initWithTitle:@"首页" image:mainImage selectedImage:mainImageSel];
  mainNav.tabBarItem = mainTabBarItem;
  //初始化tabBar
  UITabBarController *tabBarController = [[UITabBarController alloc] init];
  tabBarController.viewControllers = @[mainNav,nextNav];
  // 设为根ViewController
  self.window.rootViewController = tabBarController;
}
```

为tabbar添加subview:

```objc
UIView *backview = [[UIView alloc] initWithFrame:self.tabBar.bounds];
[backview setBackgroundColor:[UIColor whiteColor]];
[self.tabBar addSubview:backview];
```

### 与视图控制器及其视图交互

**application:didFinishLaunchingWithOptions:**	该方法中设置和初始化应用窗口的根视图控制器。该方法只会在应用启动后调用一次。如果从其他应用切换回本地应用，该方法不会被调用，如果关闭应用后台在重启才会再次调用。      
**initWithNibName：bundle：**该方法是UIViewController的指定初始化方法，创建视图控制器时，就会调用该方法。  
**loadView：**覆盖该方法，使用代码方式设置视图控制的的view属性。  
**viewDidLoad：**该方法会在视图控制器加载完视图后被调用。  
**viewWillAppear：**该方法会在视图控制器的view显示在屏幕上时被调用。  



## 文本框与文本输入

### 文本框

**UIResponder** 是 UIKit框架中的一个抽象类。诸如 **UIView，UIViewController，UIApplication** 都是它的子类。 **UIResponder** 定义了一系列方法，用于接收和处理用户事件，例如触摸事件，运动时间(如摇晃)和功能控制事件(如编辑文本和播放音乐)等。

在上面的事件中，触摸事件应该由被触摸的视图负责处理，其他类型的事件会由第一响应者负责处理。 **UIWindow** 有一个 firstResponder 属性指向第一响应者。

### main() 与 UIApplication

用 c 写的程序，入口都是 **main()**:

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

**UIApplicationMain** 函数会创建一个 **UIApplication** 对象。每个 iOS 应用都有且只有一个 **UIApplication** 对象，该对象作用是维护运行循环。一旦程序创建了某个 **UIApplication** 对象，该对象就会一直循环下去， **main()** 的执行也会因此堵塞。

**UIApplicationMain** 函数还会创建某个指定类的对象，并将其设置为 **UIApplication** 对象的 delegate。该对象的类由 **UIApplicationMain** 的最后一个参数指定。

应用启动运行循环并开始接收事件前， **UIApplication** 对象会像其委托发送一个特定的消息，使应用完成相应的初始化工作。这个消息的名称是 **application:didFinishLaunchingWithOptions:**



## UIMenuController

iOS SDK 提供一个名为 **UIMenuController** 的类，可以用来显示菜单。例如在单击手势的实现方法中弹出一个菜单：

```objc
- (void)tap:(UIGestureRecognizer *)gr{
  // 获取手势的坐标
  CGPoint point = [gr locationInView:self];
  // 让视图变成 UIMenuItem 动作消息的目标
  [self becomeFirstResponder];
  // 获取 UIMenuController 对象
  UIMenuController *menu = [UIMenuController sharedMenuController];
  // 创建一个新的标题为"delete"的UIMenuItem对象
  UIMenuItem *deleteItem = [[UIMenuItem alloc] initWithTitle:@"Delete" action:@selector(deleteLine:)];
  menu.menuItems = @[deleteItem];
  
  // 先为 UIMenuController 对象设置显示区域，然后将其设置为可见
  [menu setTargetRect:CGRectMake(point.x,point.y,2,2) inView:self];
  [menu setMenuVisible:YES animated:YES];
} 
```

要显示 **UIMenuController** 对象，还要满足一个条件：显示 **UIMenuController** 对象的 **UIView** 对象必须是当前 **UIWindow** 对象的第一响应对象。如果要将某个自定义的 **UIView** 子类对象设置为第一响应对象，就必须覆盖该对象的 **canBecomeFirstResponder** 方法：

```objc
- (BOOL)canBecomeFirstResponder{
  return YES;
}
```



## 问题：

- 什么情况下会进行重绘，重绘的时机是什么时候？
- `loadView` 什么时候会被调用？
- 系统的  `loadView` 做了什么事？什么情况下要自定义 `loadView` ？要注意什么？
- 为什么要在 xib 中设置 Files’s Owner 的 Custom Class 为目标 ViewController？
- 插座变量为什么要设置为 weak？
- 为什么不能将 ViewController 设置为 View 的 Custom Class ？
- 将某个 View 设置为 Files’s Owner 的 Custom Class 或者设置为 View 的 Custom Class 在创建方式上有什么不同？
- `initWithCoder` 何时会被调用？怎么使用？
- 如何将 ViewController 的 xib 中拖拽出的一个 view 和自定义的 View 关联？此时怎么用这个自定义的 View 加载另一个 nib 文件？有什么注意点？
- 如何设置一个 UITabBarController，包括 tabbar、tabBarItem 属性？
- 如何给一个 View 设置一个 UIMenuController？对这个 View 有什么要求？












