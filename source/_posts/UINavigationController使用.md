title: UINavigationController使用方法
date: 2017/5/4 10:07:12  
categories: iOS
tags:
	- 基本控件
---

想要从底层重写一个 app，需要对 navigation 的封装进行了解。

<!--more-->

## 常用方法

### 添加导航栏

```objc
TestViewController * mainVC = [[TestViewController alloc] init];
UINavigationController * nav = [[UINavigationController alloc] initWithRootViewController:mainVC];
self.window.rootViewController = nav;
```

### push

首先是最常用的方法：

```objc
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated;
```

使用 `setViewControllers` 一次性依次压入多个控制器，最后显示最后的控制器：

```objc
UINavigationController *nav = [[UINavigationController alloc] init];
window.rootViewController = nav;
// 创建3个测试控制器
UIViewController *vc1 = [[UIViewController alloc] init];
vc1.view.backgroundColor = [UIColor blueColor];
UIViewController *vc2 = [[UIViewController alloc] init];
vc2.view.backgroundColor = [UIColor redColor];
UIViewController *vc3 = [[UIViewController alloc] init];
vc3.view.backgroundColor = [UIColor greenColor];
// 最终会显示vc3
[nav setViewControllers:@[vc1,vc2,vc3] animated:YES];
```

### pop

常用方法：

```objc
- (UIViewController *)popViewControllerAnimated:(BOOL)animated;
```

一层层返回不方便，可以直接返回到某一个控制器：

```objc
// 返回到某一个控制器
- (NSArray *)popToViewController:VC_A animated:(BOOL)animated;

// 返回到根控制器
-(NSArray *)popToRootViewControllerAnimated:(BOOL)animated;
```

那么如何获取要 pop 到的控制器呢？

```objc
/// 当前管理的所有的控制器
@property(nonatomic,copy) NSArray<__kindof UIViewController *> *viewControllers;

/// 栈顶控制器
@property(nullable, nonatomic,readonly,strong) UIViewController *topViewController;

/// 当前可见的VC，可能是topViewController，也可能是当前topViewController present(modal)出来的VC，总而言之就是可见的VC
@property(nullable, nonatomic,readonly,strong) UIViewController *visibleViewController;
```

> 注意，topViewController与visibleViewController大部分情况一样，也有可能不同

## 导航条

### 基本属性

`NaviagationItem` 来决定大部分的显示与控制。首先看看有哪些属性：

```objc
// 中间的标题文字
@property(nullable, nonatomic,copy) NSString *title;

// 中间标题视图
@property(nullable, nonatomic,strong) UIView *titleView;

// 自定义左上角的返回按钮
@property(nullable, nonatomic,strong) UIBarButtonItem *leftBarButtonItem;
```

### 自定义左侧按钮

按钮使用的是 `UIBarButtonItem` 这个类，通过 `initWithCustomView:` 方法初始化。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // 自定义导航栏左侧按钮
    UIButton * leftBtn = [UIButton buttonWithType:UIButtonTypeRoundedRect];
    leftBtn.frame = CGRectMake(0, 7, 83, 30);
    leftBtn.backgroundColor = [UIColor orangeColor];
    [leftBtn addTarget:self action:@selector(onTap) forControlEvents:UIControlEventTouchUpInside];
    UIBarButtonItem * leftItem = [[UIBarButtonItem alloc] initWithCustomView:leftBtn];
    self.navigationItem.leftBarButtonItem = leftItem;
}

// 点击事件处理
- (void)onTap {
    NSLog(@"点击了导航栏左侧按钮");
}
```

自定义右侧按钮和左侧按钮方法类似。

### 自定义中间视图

自定义中间视图比较简单，直接设置 `navigationItem` 的 `titleView`，一般直接设置 `title` 即可。



### 给系统的 leftBarButtonItem 添加 Badge

上面提到了如何自定义左侧按钮，我们可以在自定义的 Button 上添加 Badge 实现效果。如果是系统的呢？由于 leftBarButtonItem 是 `UIBarButtonItem` 的实例，而 `UIBarButtonItem` 并非继承于 `UIView`，可以推测出，`UIBarButtonItem` 上显示的 image 和 label 是隐藏的属性。

我们需要使用 runtime 查找 `UIBarButtonItem` 中视图的属性名，并且通过 KVC，拿到这个视图属性：

```objc
// UIBarButtonItem+Badge.m
#pragma mark - 获取Badge的父视图
- (UIView *)bottomView
{
    // 通过Xcode视图调试工具找到UIBarButtonItem的Badge所在父视图为:UIImageView
    UIView *navigationButton = [self valueForKey:@"_view"];
    for (UIView *subView in navigationButton.subviews) {
        if ([subView isKindOfClass:NSClassFromString(@"UIImageView")]) {
            subView.layer.masksToBounds = NO;
            return subView;
        }
    }
    return navigationButton;
}
```

经过比对，发现视图属性名为 `_view`，拿到这个属性后，对其内部视图进行遍历，找到 UIImageView 的对象，我们就可以在 UIImageView 上添加 Badge 了。

> 同样的，我们还可以通过这种方法，自定义 UITabBarItem 的 Badge

```objc
#pragma mark - 获取Badge的父视图
- (UIView *)bottomView{
    // 通过Xcode视图调试工具找到UITabBarItem原生Badge所在父视图为:UITabBarSwappableImageView
    UIView *tabBarButton = [self valueForKey:@"_view"];
    for (UIView *subView in tabBarButton.subviews) {
        if ([subView isKindOfClass:NSClassFromString(@"UITabBarSwappableImageView")]) {
            return subView;
        }
    }
    return tabBarButton;
}
```



[Demo](https://github.com/jkpang/PPBadgeView)

[参考自:iOS: 教你给 UI 控件添加 Badge(消息提醒小圆点)](http://www.jianshu.com/p/89fa23d53400)



### 控制左边返回按钮的距离

如果我们要指定左边返回按钮距离左边框的距离，可以在在返回按钮前再插入一个 `UIBarButtonItem`：

```objc
UIBarButtonItem *leftItem = [[UIBarButtonItem alloc] initWithImage:[UIImage imageNamed:@"gobackItem.png"] style:UIBarButtonItemStylePlain target:self action:@selector(backViewcontroller)];

UIBarButtonItem *fixedItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemFixedSpace target:nil action:nil];
// 设置边框距离，个人习惯设为-16，可以根据需要调节
fixedItem.width = -16;
self.navigationItem.leftBarButtonItems = @[fixedItem, leftItem];
```

通过 `leftBarButtonItems`，插入一个不响应任何时间的 `fixspace`。

### 自定义右上角按钮或多个按钮

```objc
@property(nullable, nonatomic,strong) UIBarButtonItem *rightBarButtonItem;
/// 一次设置多个按钮
@property(nullable,nonatomic,copy) NSArray<UIBarButtonItem *> *rightBarButtonItems;
```

### 设置 navigationItem 字体格式

#### 方法一：

在 `BaseNavigationViewController` 中的 `initWithRootViewController` 中设置，或者在 `viewController` 的 `viewDidLoad` 方法中：

```objc
// 字体大小19，颜色为白色
[self.navigationController.navigationBar setTitleTextAttributes:@{NSFontAttributeName:[UIFont systemFontOfSize:19],NSForegroundColorAttributeName:[UIColor whiteColor]}];
```

####  方法二：

可以通过 `[UINavigationBar appearance]` 方法，囊括了方法一中的设置的各个位置，看还可以在 `AppDelegate` 中 `didFinishLaunchingWithOptions` 设置：

```objc
 [[UINavigationBar appearance]setTitleTextAttributes:@{NSForegroundColorAttributeName:[UIColor whiteColor],NSFontAttributeName:[UIFont systemFontOfSize:18]}];
```

> 1. 这个设置和是否隐藏导航栏一样，都是全局的，无法孤立地设置某一个 ViewController。如果有一个地方需要特殊设置，那么需要在其它地方再设置回去。
> 2. 后调用的覆盖先调用的，方法一的调用时机比方法二晚，二设置的信息会被一覆盖。

## 导航栏

### 隐藏导航栏

```objc
- (void)viewWillAppear:(BOOL)animated{
    [self.navigationController setNavigationBarHidden:YES animated:YES];
}
```

> 这个方法设置之后，后面的页面的导航栏也都隐藏了，需要在后面的页面中将其设置为 NO，所以要在 viewWillAppear 方法中，每次显示页面的时候都要重新设置。

### 修改导航栏背景色

#### 错误示范

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // 无效果
    self.navigationController.navigationBar.backgroundColor = [UIColor redColor];
}
```

这个方法无法达到理想效果，上面会盖上白色的一层 `UIImageView`，必须直接设置这个 `UIImageView` 才能达到理想的效果。

#### 正确示范

##### 方法一：

```objc
[self.navigationController.navigationBar setBackgroundImage:[UIImage imageNamed:@"xxx"] forBarMetrics:UIBarMetricsDefault];
```

就像上面说的，必须要直接设置 `UIImageView`，可以自己写个纯色的 `UIImage` 也可以直接放一张图片。

> 不要想着将这个 `UIImageView` 置为 nil，就能显示出 `navigationBar` 的背景色了。`navigationBar` 不包含上面的那个状态栏(如下图所示)，因此，会露出白白的一片，只有设置 `UIImageView` 才能将这一片也遮住。

![设置imageView为nil](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/navigationbar_imageView.png?raw=true)

##### 方法二：

还是 `[UINavigationBar appearance]` 方法：

```objc
[[UINavigationBar appearance] setBarTintColor:[UIColor redColor]];
```

一步达到效果。

## 小技巧

### 操作 navigationItem

上面我们看到操作 navigationBar 都是通过 `self.navigationController.navigationBar`，操作 navigationItem 都是通过 `self.navigationItem`。

事实上，UINavigationController 并没有`navigationItem`这样一个直接的属性，由于 UINavigationController 继承于 UIViewController ,而 UIViewController 是有`navigationItem`这个属性的是，所以对 navigationController 使用点语法获取 `navigationItem` 是编译得过的，但是这样操作是没有效果的。

> NavigationItem 是一个 NSObject 对象，里面保存了导航栏上的各个 view；
>
> NavigationBar 是一个 UIView 对象，NavigationItem 中的各个 view 都被添加到其上





### UINavigationController 返回手势失效

系统为 UINavigationController 提供了一个 `interactivePopGestureRecognizer` 用于右滑返回(pop),但是，如果自定了 back button 或者隐藏了 navigationBar ，该手势就失效了。我们需要自己实现一下 delegate 方法;

新建一个 `BaseNavigationController` 实现 delegate：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    self.interactivePopGestureRecognizer.delegate =  self;
}
```

需要实现的代理方法：

```objc
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer {
    if (self.viewControllers.count <= 1 ) {
        return NO;
    }

    return YES;
}
```

考虑到在 push 动画发生的时候，要禁止滑动手势，所以继续在 `BaseNavigationController` 中实现代理方法，禁止滑动手势：

```objc
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    [super pushViewController:viewController animated:animated];
    self.interactivePopGestureRecognizer.enabled = NO;
}
```

然后在新 push 出来的 ViewController 中设置启用滑动手势：

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    self.navigationController.interactivePopGestureRecognizer.enabled = YES;
}
```

