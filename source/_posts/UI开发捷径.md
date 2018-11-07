title: 《iOS UI 开发捷径》读书笔记
date: 2016/7/31 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

读了 《iOS UI 开发捷径》，书上的东西比较简单，也比较零碎。所以做个大致的整理。

<!--more-->

### xib 加载 Cell

有一个特殊的 View 是 cell:

```objc
// 平时使用 loadNibNamed:owner:options 方法会先把 xib 加载到内存生成相应的 nib，再通过 nib 转化为 View 实例。对于 UITableView 来说，重复加载 xib 是一个耗时操作，所以 UITableView 通过注册 UINib 来缓存 nib
[_tableView registerNib:[Nib nibWithNibNamed:@"SomeView"] bundle:nil];
```

### xib 加载 VC

可以通过以下方式创建于 xib 关联的 VC:

```swift
// 有同名的 xib 文件
let homeVC = HomeViewController()
// 没有同名的 xib 文件
let homeVC = HomeViewController.init(nibName:"HomeView", bundle:nil)
```

### 加载 bundle 中的资源

应用的代码和资源都放在应用所在的 main bundle 中，不过有一些第三方的 sdk 会把自己的资源文件打包成一个 bundle。这些 bundle 也会保存在 main bundle 中，我们可以先拿到其 bundle 再拿资源文件：

```swift
let subBundle = Bundle.init(path: Bundle.main.path(forResource: "myBundle" ofType:"bundle")!)
let myView = subBundle?.loadNibNamed("BannerView", owner:nil, options:nil)?[0] as! UIView
```

### 加载 image 方式

有两种方式加载 image：

```swift
let image = UIImage.init(named:"someImage")
let image = UIImage.init(contentsOfFile:(Bundle.main.path(forResource:"someImage", ofType:"png"))!)
```

第一种默认加载 mainbundle 中的图片，**Assets.xcassets** 中的图片只能通过第一种方式加载

第二种可以加载任意位置的图片。第一种加载的图片会被系统缓存。所以显示 icon 的时候通过第一种方式加载比较好，图片资源较大时，用第二种方式

### 如何删除 storyboard

storyboard的入口在**targets->General->Deployment Info->Main Interface**，默认的值为Main，即初始默认storyboard的名字。因此，想要不加载storyboard，需要将这个默认值删掉，就不会再从storyboard进入了。

使用storyboard的时候 `application:didFinishWithOptions:` 方法只返回一个YES。删除后，需要添加代码对window初始化，否则app什么也显示不了。

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
	ViewController *uv = [[ViewController alloc] initWithNibName:@"ViewController" bundle:nil];
	[self.window setRootViewController:uv];
	[self.window makeKeyAndVisible];
	return YES;
}
```

### 拉伸图片

聊天的气泡图可以通过代码的方式设置，也可以直接在 *Assets.xcassets* 文件夹内直接设置。选中相应图片，找到面板中的 **show slicing** 点击，会出现相应的设置相关项（用到时候查一下，使用起来非常简单）。

### 使用 @IBOutlet 设置约束

代码中声明约束变量：

```swift
@IBOutlet weak var widthConstraint: NSLayoutConstraint!
```

在 xib 中右键然后和代码中声明的变量连线。必要的时候，使用动画改变约束：

```swift
UIView.animated(withDuration:0.5, animations:{
    self.widthConstraint.constant = 300
    self.view.layoutIfNeeded()
})
```

### User Defined Runtime Atrributes

使用User Defined Runtime Attributes可以配置一些在interface builder 中不能配置的属性。比如写一个button的时候经常会需要设置其 `cornerRadious`，`borderWidth` 和 `borderColor` 三个属性:

```
Key.Path						Type
layer.cornerRadius				Number
layer.borderWidth				Number
layer.borderColor				Color
```

但是，进过设置后会发现 `borderColor` 属性设置并不成功。这是因为这里设置的颜色类型是 `UIColor` 而 `borderColor` 是 `CGColor` 因此显示不出来。

需要定义一个 `CALayer` 的分类:

```objc
#import "CALayer+Additions.h"
#import <UIKit/UIKit.h>
@implementation CALayer (Additions)
- (void)setBorderColorFromUIColor:(UIColor *)color{
    self.borderColor = color.CGColor;
}
@end
```

xib中的Key.Path 改为此属性 **layer.borderColorFromUIColor** 即可通过 runtime 动态设置。





### storyboard 全面学习

- sb 中可以向一个 tableViewController 添加一个或多个 Table View Cell。可以设置其不同的 Identifier。
- Extra View 可以为 VC 提供一个额外的关联视图。你不能将 View 拖到 VC 中，而是应该和 VC 平级。你可以再将这个 View 和 代码关联，在需要的时候添加到 VC 中。
- sb 中的 segue 用 Identifier 区分。Class 是 segue 的类型，一般为 UIStoryboardSegue
- 可以重写 `prepare(for:sender:)` 方法。通过 Identifier 找到对应的 segue，从而确定跳转的页面关系，`segue.destination` 就是目标 VC。
- 可以重写 `shouldPerformSegue(withIdentifier:sender:)` 来判断是否需要跳转。
- 可以调用 `performSegue(withIdentifier:sender:)` 来自己触发跳转

 

