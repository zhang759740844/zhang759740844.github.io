title: 删除Storyboard
date: 2016/8/11 14:07:12  
categories: IOS
tags: [xib]

---

xcode版本升级后，storyboard成了ios新建项目后默认的布局。但是尝试了一会后发觉很不习惯，于是研究了下删除storyboard的方法。
<!--more-->

storyboard的入口在**targets->General->Deployment Info->Main Interface**，默认的值为Main，即初始默认storyboard的名字。因此，想要不加载storyboard，需要将这个默认值删掉，就不会再从storyboard进入了。

删除后，viewController需要一个布局，需要手动创建一个xib文件。需要对xib文件和viewcontroller进行关联。
- 点击placeholders下的File's Owner将Custom Class的Class设置为viewcController的名字。
- 将File's Owner的view和xib的View进行关联。

使用storyboard的时候application:didFinishWithOptions:方法只返回一个YES。删除后，需要添加代码对window初始化，否则app什么也显示不了。
```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
	UIViewController *uv = [[UIViewController alloc] initWithNibName:@"ViewController" bundle:nil];
	[self.window setRootViewController:uv];
	[self.window makeKeyAndVisible];
	return YES;
}
```
这里要使用initWithNibName方法。

至此，app就可以显示新建的xib文件的布局了
