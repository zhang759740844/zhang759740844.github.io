title: UINavigationController使用方法
date: 2017/5/4 10:07:12  
categories: iOS
tags:

	- 基本控件
---

navigation 简单了解下使用

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

### 设置默认返回按钮

#### 设置返回箭头

导航栏默认有一个返回的按钮，我们可以自定义它的箭头：

```objc
// 设置颜色时，imageWithRenderingMode 设置 UIImage 渲染为原来的颜色
[[UINavigationBar appearance] setBackIndicatorImage:[[UIImage imageNamed:@"back"] imageWithRenderingMode:UIImageRenderingModeAlwaysOriginal]];

// 使用 tintColor 设置颜色
[[UINavigationBar appearance] setBackIndicatorImage:[UIImage imageNamed:@"back"]];
[[UINavigationBar appearance] setBackIndicatorTransitionMaskImage:[UIImage imageNamed:@"back"]];
[[UINavigationBar appearance] setTintColor:[UIColor lightGrayColor]];
```

#### 通用的隐藏返回文字

返回按钮旁的标题默认是上一级页面的 title，可以如下设置隐藏 title:

```objc
[[UIBarButtonItem appearance] setBackButtonTitlePositionAdjustment:UIOffsetMake(-100, 0) forBarMetrics:UIBarMetricsDefault];
```

#### 特殊页面设置返回按钮

把自定义的 barbutton 设置到 `backBarButtonItem` 即可

```swift
let backbtn = UIBarButtonItem(title: "取消", style: UIBarButtonItemStyle.Plain, target:self, action: nil)
self.navigationItem.backBarButtonItem = backbtn
```

> 注意，这段代码要写在上一个控制器中。因为 back 要返回的是上一个控制器。

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

> 注意，设置了 leftBarButtonItem 就替代了原来的的返回按钮。可以通过设置 `self.navigationItem.leftItemsSupplementBackButton = YES;` 保留返回按钮和 leftItems

### 自定义中间视图

自定义中间视图比较简单，直接设置 `navigationItem` 的 `titleView`。



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

### 自定义右上角按钮或多个按钮

```objc
@property(nullable, nonatomic,strong) UIBarButtonItem *rightBarButtonItem;
/// 一次设置多个按钮
@property(nullable,nonatomic,copy) NSArray<UIBarButtonItem *> *rightBarButtonItems;
```

### 设置 navigationItem 字体格式

可以通过 `[UINavigationBar appearance]` 方法 设置：

```objc
 [[UINavigationBar appearance]setTitleTextAttributes:@{NSForegroundColorAttributeName:[UIColor whiteColor],NSFontAttributeName:[UIFont systemFontOfSize:18]}];
```

> 这个设置和是否隐藏导航栏一样，都是全局的，无法孤立地设置某一个 ViewController。如果有一个地方需要特殊设置，那么需要在其它地方再设置回去。

### 操作 navigationItem

上面我们看到操作 navigationBar 都是通过 `self.navigationController.navigationBar`，操作 navigationItem 都是通过 `self.navigationItem`。

事实上，UINavigationController 并没有`navigationItem`这样一个直接的属性，由于 UINavigationController 继承于 UIViewController ,而 UIViewController 是有`navigationItem`这个属性的是，所以对 navigationController 使用点语法获取 `navigationItem` 是编译得过的，但是这样操作是没有效果的。

> NavigationItem 是一个 NSObject 对象，里面保存了导航栏上的各个 view；
>
> NavigationBar 是一个 UIView 对象，NavigationItem 中的各个 view 都被添加到其上

### UINavigationController 返回手势失效

系统为 UINavigationController 提供了一个 `interactivePopGestureRecognizer` 用于右滑返回(pop),但是，如果自定了 left button 或者隐藏了 navigationBar ，该手势就失效了。我们需要自己实现一下 delegate 方法;

> 不过一般我们最好还是使用默认的 backbutton 返回为好。一般自定义了 left button，并且隐藏了 backbutton 的情况，都应该是不让用户能够直接返回的情况。

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

## 导航栏

### 隐藏导航栏

```objc
- (void)viewWillAppear:(BOOL)animated{
    [self.navigationController setNavigationBarHidden:YES animated:animated];
}
```

> 这个方法设置之后，后面的页面的导航栏也都隐藏了。所以要注意一个原则，在当前页面 `viewWillAppear` 方法中做的修改，也要在当前页面的 `viewWillDisappear` 中还原回来。

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

```objc
[self.navigationController.navigationBar setBackgroundImage:[UIImage imageNamed:@"xxx"] forBarMetrics:UIBarMetricsDefault];
```

就像上面说的，必须要直接设置 `UIImageView`，可以自己写个纯色的 `UIImage` 也可以直接放一张图片。

### 设置导航栏透明度

设置导航栏透明度需要自己创建一个带有透明度的 UIImage:

```objc
// 背景色
UIImage *image = [self imageWithColor:[color colorWithAlphaComponent:alpha]];
[self.navigationController.navigationBar setBackgroundImage:image forBarMetrics:UIBarMetricsDefault];

- (UIImage *)imageWithColor:(UIColor *)color {
    CGRect rect = CGRectMake(0, 0, 1, 1);
    UIGraphicsBeginImageContext(rect.size);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(context, [color CGColor]);
    CGContextFillRect(context, rect);
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}
```

> 这里只创建了一个像素的图片，但是还是能达到平铺整个导航栏的效果。
>
> 创建的 UIImage 的大小只有一个像素的好处是节约 CPU 资源。

### 隐藏导航栏底部的分割线

就是设置一个透明的图片，和设置导航栏背景色一个道理。

```objc
UINavigationBar *navigationBar = self.navigationController.navigationBar;
//此处使底部线条失效
[navigationBar setShadowImage:[UIImage new]];
```

### 导航栏的几个属性

这些属性一般用处不大，但是有点印象就行了。以下为默认值

```objc
self.navigationController.navigationBar.translucent = YES;
self.edgesForExtendedLayout = UIRectEdgeAll;
self.automaticallyAdjustsScrollViewInsets = YES;
self.extendedLayoutIncludesOpaqueBars = NO;
```

- 导航半透明的时候
  -  设置 `edgesForExtendedLayout` 为 `UIRectEdgeAll` ，视图会延伸到导航栏下。默认为 YES
    - 此时默认 `automaticallyAdjustScrollViewInsets` 为 YES。ScrollView 会自动设置 contentInset
  -  设置`edgesForExtendedLayout` 为 `UIRectEdgeNone` ，视图不会延伸。
- 导航栏不透明的时候
  - 设置 `edgesForExtendedLayout` 无效。
  - 设置 `extendedLayoutIncludesOpaqueBars` 为 YES，可以延伸到不透明的导航栏下
  - 设置`extendedLayoutIncludesOpaqueBars`  为 NO，不会延伸到透明的导航栏系下，默认为 NO

### 更改顶部状态栏颜色

1. 在工程的Info.plist文件中添加一行**UIViewControllerBasedStatusBarAppearance**，选择Boolean类型，并设置为YES，Xcode会自动把名称变为View controller-based status bar appearance。

2. 在你的ViewController中添加下面的方法

   ```objc
   -(UIStatusBarStyle)preferredStatusBarStyle{
       // return UIStatusBarStyleDefault; 黑色
       return UIStatusBarStyleLightContent; // 白色
   }
   ```

3. 调用 UIViewController 的 `setNeedsStatusBarAppearanceUpdate` 方法，通知状态栏颜色改变了。

### 全屏滑动返回

实现全屏滑动返回仅需在导航栏给导航栏添加`UIGestureRecognizerDelegate`协议，并在ViewDidLoad中设置。关键在于调用系统返回处理方法 `hyandleNavigationTransition`

```objc
// 获取系统自带滑动手势的target对象
id target = self.interactivePopGestureRecognizer.delegate;

// 创建全屏滑动手势，调用系统自带滑动手势的target的action方法
UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] initWithTarget:target action:@selector(handleNavigationTransition:)];

// 设置手势代理，拦截手势触发
pan.delegate = self;

// 给导航控制器的view添加全屏滑动手势
[self.view addGestureRecognizer:pan];

// 禁止使用系统自带的滑动手势
self.interactivePopGestureRecognizer.enabled = NO;
```

### 导航栏过渡

导航栏过渡是一个需要好好设计的功能。如果做得不好，push 和 pop 时候会非常僵硬。

比较好的方式是通过 Method Swizzling 获取系统方法 `_updateInteractiveTransition` 拿到当前的进度。

具体可参考如下两篇文章：

[iOS: 记一次导航栏平滑过渡的实现](https://www.jianshu.com/p/859a1efd2bbf)

[超简单！！！ iOS设置状态栏、导航栏按钮、标题、颜色、透明度，偏移等](https://www.jianshu.com/p/540a7e6f7b40)

如果要求不高，对于特殊的页面，直接在 `viewWillAppear` 和 `viewWillDisapper` 中通过 `[self.navigationController setNavigationBarHidden:YES animated:YES]` 显示和隐藏即可。