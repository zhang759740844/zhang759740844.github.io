title: app 间的跳转与信息传递
date: 2017/7/2 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

日常使用中，经常会遇到分享、支付等功能需要跳转到另一个 app 进行处理，然后处理回调信息。本篇将对这个功能进行学习。

<!--more-->

应用间跳转是通过苹果实现的类似于路由的 URL Schemes（自定义协议头）实现的。应用B约定好打开自身的协议头，应用A则在特定的场合调用这个协议头，实现跳转。

## URL Scheme

### A->B

#### 直接跳转

##### B中注册

要跳转到B，就得先让B注册自己的 URL Schemes。在 Info->URL Types 中新建一个:

![注册 URL Schemes](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/schemes_1.png?raw=true)

这里的添加最终映射到的是 plist 文件中，URL Schemes 将在 plist 文件中以键值对的方式保存，可以自由的添加和删除

![plist](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/schemes2.png?raw=true)

##### A中调用

B中注册好了 URL Schemes，我们**可以直接在 safari 中输入 `zacharyB1://` 就可以打开 app 了，这也是很多h5中要打开关联的 app 都提示在 safari 中打开的原因**。我们现在在A中添加跳转逻辑：

```objc
- (IBAction)jumpToAppB:(id)sender {
   // 1.获取应用程序App-B的URL Scheme
   NSURL *appBUrl = [NSURL URLWithString:@"zacharyB1://"];

   // 2.判断手机中是否安装了对应程序
   if ([[UIApplication sharedApplication] canOpenURL:appBUrl]) {
       // 3. 打开应用程序App-B
       [[UIApplication sharedApplication] openURL:appBUrl];
   } else {
       NSLog(@"没有安装");
   }
}
```

这里要注意一点，iOS9 后引入了白名单的概念。A在调用 `canOpenURL:` 方法的时候，会在A的白名单中查看是否可以跳转应用B，所以我们还要提前将B注册到A的白名单当中。具体做法就是在应用A的 plist 文件中，添加 `LSApplicationQueriesSchemes` 数组，然后减价键值为 `zacharyB1` 的字符串：

![白名单](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/schemes3.png?raw=true)

添加好白名单后，就可以正常的跳转了。

#### 带参数的跳转

##### A中调用

上面是直接跳转打开应用B，但是实际情况下，我们可能要打开B的某个页面，并且要传递一些参数过去。本身 `openURL` 是不带参数的，我们需要按照约定的规则，将参数作为 URL 的一部分传入，由应用B自己解析。

比如区分跳转到B的某一个页面1还是页面2，我们分别定制 URL：

```objc
- (IBAction)jumpToAppBPage1:(id)sender {
    // 1.获取应用程序App-B的Page1页面的URL
    NSURL *appBUrl = [NSURL URLWithString:@"zacharyB1://Page1"];

    // 2.判断手机中是否安装了对应程序
    if ([[UIApplication sharedApplication] canOpenURL:appBUrl]) {
        // 3. 打开应用程序App-B的Page1页面
        [[UIApplication sharedApplication] openURL:appBUrl];
    } else {
        NSLog(@"没有安装");
    }
}

- (IBAction)jumpToAppBPage2:(id)sender {
    // 1.获取应用程序App-B的Page2页面的URL
    NSURL *appBUrl = [NSURL URLWithString:@"zacharyB1://Page2"];

    // 2.判断手机中是否安装了对应程序
    if ([[UIApplication sharedApplication] canOpenURL:appBUrl]) {
        // 3. 打开应用程序App-B的Page2页面
        [[UIApplication sharedApplication] openURL:appBUrl];
    } else {
        NSLog(@"没有安装");
    }
}
```

##### B中处理

在A调起B后，B中会执行特定方法，我们可以在其中添加处理逻辑，获取参数：

```objc
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
    // 1.获取导航栏控制器
    UINavigationController *rootNav = (UINavigationController *)self.window.rootViewController;

    // 2.每次跳转前必须是在跟控制器(细节)
    [rootNav popToRootViewControllerAnimated:NO];   

    // 3.根据字符串关键字来跳转到不同页面
    if ([url.absoluteString containsString:@"Page1"]) { // 跳转到应用B的Page1页面
		[rootNav pushViewController:[[Page1ViewController alloc] init] animated:YES];
    } else if ([url.absoluteString containsString:@"Page2"]) { // 跳转到应用B的Page2页面
        [rootNav pushViewController:[[Page1ViewController alloc] init] animated:YES];
    }   

    return YES;
}
```

### B->A

B返回A的过程和A调用B的过程类似，相当于B主动调用A的 URL Scheme。B如何知道是哪个应用调用了它呢？这就需要A将自己的 URL Scheme 传给B。同样的，需要约定规则，比如如下：

```
应用B的URL Schemes://要传的参数?应用A的URL Schemes
```

将应用A的 URL Scheme 在末尾传入，并且以 `?` 分隔。B拿到后取出A的 URL Scheme，在必要的时候（如点击某个按钮）通过其调起应用A。

```objc
- (IBAction)page1BackToAppA:(id)sender {
   // 1.拿到对应应用程序的URL Scheme
   NSString *urlSchemeString = [[self.urlString componentsSeparatedByString:@"?"] lastObject];
   NSString *urlString = [urlSchemeString stringByAppendingString:@"://"];

   // 2.获取对应应用程序的URL
   NSURL *url = [NSURL URLWithString:urlString];

   // 3.判断是否可以打开
   if ([[UIApplication sharedApplication] canOpenURL:url]) {
       [[UIApplication sharedApplication] openURL:url];
   }
}
```

## Universal Links

除了 URL Scheme，iOS9中还提供了一种系统级的 APP 唤醒方式。这可以使你的应用可以通过传统的HTTP链接来启动APP(如果iOS设备上已经安装了你的app，不管在微信里还是在哪里)， 或者打开网页(iOS设备上没有安装你的app)。

### 开启使用

首先要在开发者中心中开启这一功能：

![开启](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/schemes4.png?raw=true)

然后在项目的配置中找到 Capabilities->Associated Domains：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/schemes5.png?raw=true)

以 `applinks:` 开头，后面跟着你的域名。这个域名必须支持 HTTPS 的 GET 请求。在第一次启动 APP 的时候，系统会自动到这个域名的根目录下，请求名为**apple-app-site-association**的文件，即会发送 ` https://domain.com/apple-app-site-association ` 请求。

###  apple-app-site-association 格式

**apple-app-site-association** 是一个内容为json文件，且没有后缀名的文件(不能带.json的后缀)。文件内容大致如下：

```json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "9JA89QQLNQ.com.apple.wwdc",
                "paths": [ "/wwdc/news/", "/videos/wwdc/2015/*"]
            },
            {
                "appID": "ABCD1234.com.apple.wwdc",
                "paths": [ "*" ]
            }
        ]
    }
}
```

appID 的组成部分为：`TeamID . BundleId TeamID`。其中，`TeamID` 可以从苹果开发账号页面也 Your Account 下查看，BundleId 就直接在工程里看了。

完成这步后，系统就将域名和 app 绑定了起来。在任何其它应用中加载这个域名下的链接都将自动打开你绑定的 app。

### APP 内的处理

打开 APP 后会调用以下方法：

```objc
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler{
    if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]){
      	//url是完整的带 domain 的url
        NSURL *url = userActivity.webpageURL;
        if (url是我们希望处理的){
            //打开对应页面
        }
        else{
          	//不能识别的用 safari 打开
            [[UIApplication sharedApplication] openURL:url];
        }
    }

    return YES;
}
```

当 `userActivity` 是 `NSUserActivityTypeBrowsingWeb` 类型, 则意味着它已经由通用链接 API 代理。这样的话, 它保证用户打开的 URL 将有一个非空的 `webpageURL` 属性。



[iOS开发--一步步教你彻底学会『iOS应用间相互跳转』](http://www.jianshu.com/p/b5e8ef8c76a3)

[iOS 9学习系列：打通 iOS 9 的通用链接（Universal Links）](http://www.cocoachina.com/ios/20150902/13321.html)

