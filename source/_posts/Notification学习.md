title: Notification 使用小结
date: 2017/4/3 14:07:12  
categories: iOS
tags: 
	- 学习笔记


------

最近需求有关于通知的，折腾了许久，现在集中学习一下。主要针对 iOS10 的新特性。

<!--more-->

## 基本原理

### 本地推送（Local Notification）

1. App 本地创建通知，加入系统的 Schedule 中。
2. 如果达成触发条件则推送相应消息内容。

### 远程推送（Remote Notification）

1. 设备向 APNS(Apple Push Notification Service)发送请求，注册自己。APNS 返回一个 `deviceToken`，设备再将发送给 PUSH 服务器。
2. 服务器将要发送的消息，目标设备的 `deviceToken` 打包，发给 APNS。
3. APNS 在自身的已注册 Push 服务的 iPhone 列表中，查找有相应标识的iPhone，并把消息发到 iPhone。
4. iPhone 把发来的消息传递给相应的应用程序， 并且按照设定弹出 Push 通知。

## 基本设置

### 配置

关于推送证书的准备就不说了。除了推送证书，还需要打开 Push Notification 开关进行授权。打开开关后，会自动生成一个 xxx.entitlements 文件，如下图所示：

![entitlement](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/push_entitlement.png?raw=true)

这里的 value 默认是 `development`，不需要我们手动改，Xcode 会在我们发布的时候自动改成 `producetion` 的。

###  方法

#### 头文件

导入 `#import <UserNotifications/UserNotifications.h>`
且要遵守`<UNUserNotificationCenterDelegate>`的协议，在 Appdelegate.m中。这里需要注意，我们最好写成这种形式（防止低版本找不到头文件出现问题）:

```objc
#ifdef NSFoundationVersionNumber_iOS_9_x_Max
#import <UserNotifications/UserNotifications.h>
#endif
```

#### 获取权限与注册

在 Appdelegate 中先向用户获取权限与注册：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [self replyPushNotificationAuthorization:application];
    return YES;
}

#pragma mark - 申请通知权限
// 申请通知权限
- (void)replyPushNotificationAuthorization:(UIApplication *)application{

    if (IOS10_OR_LATER) {
        //iOS 10 later
      	//获取单例对象
        UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
        //必须写代理，不然无法监听通知的接收与点击事件
        center.delegate = self;
        [center requestAuthorizationWithOptions:(UNAuthorizationOptionBadge | UNAuthorizationOptionSound | UNAuthorizationOptionAlert) completionHandler:^(BOOL granted, NSError * _Nullable error) {
            if (!error && granted) {
                //用户点击允许
                NSLog(@"注册成功");
            }else{
                //用户点击不允许
                NSLog(@"注册失败");
            }
        }];

        // 可以通过 getNotificationSettingsWithCompletionHandler 获取权限设置
        //之前注册推送服务，用户点击了同意还是不同意，以及用户之后又做了怎样的更改我们都无从得知，现在 apple 开放了这个 API，我们可以直接获取到用户的设定信息了。注意UNNotificationSettings是只读对象哦，不能直接修改！
        [center getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings) {
            NSLog(@"========%@",settings);
        }];
    }else if (IOS8_OR_LATER){
        //iOS 8 - iOS 10系统
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:UIUserNotificationTypeAlert | UIUserNotificationTypeBadge | UIUserNotificationTypeSound categories:nil];
        [application registerUserNotificationSettings:settings];
    }
  	//向APNS注册
  	[application registerForRemoteNotifications];
}
```

其中定义的宏：

```objc
#define IOS10_OR_LATER ([[[UIDevice currentDevice] systemVersion] floatValue] >= 10.0)
#define IOS9_OR_LATER ([[[UIDevice currentDevice] systemVersion] floatValue] >= 9.0)
#define IOS8_OR_LATER ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.0)
#define IOS7_OR_LATER ([[[UIDevice currentDevice] systemVersion] floatValue] >= 7.0)
```

上面方法兼顾了 iOS8 以上的各个系统。调用了上面的方法后，真机就会在屏幕上显示获取通知权限的对话框。

通过 `getNotificationSettingsWithCompletionHandler:` 这个新方法可以获取用户的设定信息，打印信息如下：

```objc
 [center getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings) {
            NSLog(@"========%@",settings);
  }];
打印信息如下：
========<UNNotificationSettings: 0x1740887f0; authorizationStatus: Authorized, notificationCenterSetting: Enabled, soundSetting: Enabled, badgeSetting: Enabled, lockScreenSetting: Enabled, alertSetting: NotSupported, carPlaySetting: Enabled, alertStyle: Banner>
```

> 1. UNUserNotificationCenter 是一个单例的对象，可以在代码的任意位置获取该对象的实例，请求通知权限。
> 2. DeviceToken可能会有更改，因此需要在程序的每次启动都去注册和设置。

#### 获取 deviceToken

上面的方法向苹果注册后，苹果返回 `deviceToken`:

```objc
#pragma  mark - 获取device Token
//获取DeviceToken成功
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken{
    NSLog(@"deviceToken===========%@",deviceString);
  // 将 deviceToken 发送给各种第三方推送或者自己的服务端
  ...
}

//获取DeviceToken失败
- (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error{
    NSLog(@"[DeviceToken Error]:%@\n",error.description);
}
```

#### 接收通知

##### 通知的类型

在 iOS10 中，本地通知和远程通知合二为一。区分本地通知跟远程通知的类是`UNPushNotificationTrigger.h`类中，`UNPushNotificationTrigger`的类型是新增加的，通过它，我们可以得到一些通知的触发条件 ，解释如下：

1. `UNPushNotificationTrigger` （远程通知） 远程推送的通知类型
2. `UNTimeIntervalNotificationTrigger` （本地通知） 一定时间之后，重复或者不重复推送通知。我们可以设置timeInterval（时间间隔）和repeats（是否重复）。
3. `UNCalendarNotificationTrigger`（本地通知） 一定日期之后，重复或者不重复推送通知 例如，你每天8点推送一个通知，只要dateComponents为8，如果你想每天8点都推送这个通知，只要repeats为YES就可以了。
4. `UNLocationNotificationTrigger` （本地通知）地理位置的一种通知，当用户进入或离开一个地理区域来通知。

##### iOS10

对于 iOS10，且实现了 `UNUserNotificationCenterDelegate` 协议，将通过下面两个方法接收和处理通知：

当App处于前台：

```objc
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler{

    //收到推送的请求
    UNNotificationRequest *request = notification.request;

    //收到推送的内容
    UNNotificationContent *content = request.content;

    //收到用户的基本信息
    NSDictionary *userInfo = content.userInfo;

    //收到推送消息的角标
    NSNumber *badge = content.badge;

    //收到推送消息body
    NSString *body = content.body;

    //推送消息的声音
    UNNotificationSound *sound = content.sound;

    // 推送消息的副标题
    NSString *subtitle = content.subtitle;

    // 推送消息的标题
    NSString *title = content.title;

    if([notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
        //此处省略一万行需求代码。。。。。。
        NSLog(@"iOS10 收到远程通知:%@",userInfo);

    }else {
        // 判断为本地通知
        //此处省略一万行需求代码。。。。。。
        NSLog(@"iOS10 收到本地通知:{\\\\nbody:%@，\\\\ntitle:%@,\\\\nsubtitle:%@,\\\\nbadge：%@，\\\\nsound：%@，\\\\nuserInfo：%@\\\\n}",body,title,subtitle,badge,sound,userInfo);
    }


    // 需要执行这个方法，选择是否提醒用户，有Badge、Sound、Alert三种类型可以设置
    completionHandler(UNNotificationPresentationOptionBadge|
                      UNNotificationPresentationOptionSound|
                      UNNotificationPresentationOptionAlert);

}
```

这里就可以通过上面列举的类型判断推送的类型了。最后一个执行的 `completionHandler(...)` 方法可以在前台下也显示横幅。

App 横幅的点击事件：

```objc
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler{
    //收到推送的请求
    UNNotificationRequest *request = response.notification.request;

    //收到推送的内容
    UNNotificationContent *content = request.content;

    //收到用户的基本信息
    NSDictionary *userInfo = content.userInfo;

    //收到推送消息的角标
    NSNumber *badge = content.badge;

    //收到推送消息body
    NSString *body = content.body;

    //推送消息的声音
    UNNotificationSound *sound = content.sound;

    // 推送消息的副标题
    NSString *subtitle = content.subtitle;

    // 推送消息的标题
    NSString *title = content.title;

    if([response.notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
        NSLog(@"iOS10 收到远程通知:%@",userInfo);
        //此处省略一万行需求代码。。。。。。

    }else {
        // 判断为本地通知
        //此处省略一万行需求代码。。。。。。
        NSLog(@"iOS10 收到本地通知:{\\\\nbody:%@，\\\\ntitle:%@,\\\\nsubtitle:%@,\\\\nbadge：%@，\\\\nsound：%@，\\\\nuserInfo：%@\\\\n}",body,title,subtitle,badge,sound,userInfo);
    }
  
    completionHandler(); // 系统要求执行这个方法
}
```

这个方法的名字是 `didReceiveNotificationResponse`，是把本地推送和远端推送整合在一起了。下面👇的方法就是专门针对远端推送的。

##### iOS10 前

app 在前台，不会有通知条幅，但是还是能走到下面的方法。进程杀死，点击条幅启动后也能走到下面的方法。

```objc
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    NSLog(@"iOS7及以上系统，收到通知:%@", userInfo);
    completionHandler(UIBackgroundFetchResultNewData);
    //此处省略一万行需求代码。。。。。。
}
```

> 如果iOS10，且实现了上面的代理方法，则不会走下面的这个方法。
>
> 无论进程是否被杀死，都能走到上面的方法中。

### 添加推送

#### 本地推送

本地推送的基本流程是：

1. 创建一个触发器（trigger）
2. 创建推送的内容（UNMutableNotificationContent）
3. 创建推送请求（UNNotificationRequest）
4. 推送请求添加到推送管理中心（UNUserNotificationCenter）中

##### 创建触发器（trigger）

**新功能trigger**可以在特定条件触发，有三类: `UNTimeIntervalNotificationTrigger`、`UNCalendarNotificationTrigger`、`UNLocationNotificationTrigger`。

定时推送：

```objc
//timeInterval：单位为秒（s）  repeats：是否循环提醒
//50s后提醒
UNTimeIntervalNotificationTrigger *trigger1 = [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:50 repeats:NO];
```

定期推送：

```objc
//在每周一的14点3分提醒
NSDateComponents *components = [[NSDateComponents alloc] init]; 
components.weekday = 2;
components.hour = 16;
components.minute = 3;
 // components 日期
UNCalendarNotificationTrigger *calendarTrigger = [UNCalendarNotificationTrigger triggerWithDateMatchingComponents:components repeats:YES];
```

地点推送：

地区信息使用CLRegion的子类**CLCircularRegion**，可以配置region属性 **notifyOnEntry**和**notifyOnExit**，是在进入地区、从地区出来或者两者都要的时候进行通知:

```objc
  //首先得导入#import <CoreLocation/CoreLocation.h>，不然会regin创建有问题。
  // 创建位置信息
  CLLocationCoordinate2D center1 = CLLocationCoordinate2DMake(39.788857, 116.5559392);
  CLCircularRegion *region = [[CLCircularRegion alloc] initWithCenter:center1 radius:500 identifier:@"经海五路"];
  region.notifyOnEntry = YES;
  region.notifyOnExit = YES;
  // region 位置信息 repeats 是否重复 （CLRegion 可以是地理位置信息）
  UNLocationNotificationTrigger *locationTrigger = [UNLocationNotificationTrigger triggerWithRegion:region repeats:YES];
```

##### 创建推送的内容（UNMutableNotificationContent）

属性有 title、subtitle、body、badge、sound、lauchImageName、userInfo、attachments、categoryIdentifier、threadIdentifier。

```objc
	// 创建通知内容 UNMutableNotificationContent, 注意不是 UNNotificationContent ,此对象为不可变对象。
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"Dely 时间提醒 - title";
    content.subtitle = [NSString stringWithFormat:@"Dely 装逼大会竞选时间提醒 - subtitle"];
    content.body = @"Dely 装逼大会总决赛时间到，欢迎你参加总决赛！希望你一统X界 - body";
    content.badge = @666;
    content.sound = [UNNotificationSound defaultSound];
    content.userInfo = @{@"key1":@"value1",@"key2":@"value2"};
```

> `categoryIdentifier` 后面会用到,就是下面 Notification Actions 里面的 identifier.

##### 创建推送请求（UNNotificationRequest）

```objc
// 创建通知标示
NSString *requestIdentifier = @"Dely.X.time";
// 创建通知请求 UNNotificationRequest 将触发条件和通知内容添加到请求中
UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:requestIdentifier content:content trigger:timeTrigger];
```

这里的 `identifier` 是用来标识本地推送的，用在对本地推送的修改删除，一般用处不是太大。

#####  添加到推送管理中心（UNUserNotificationCenter）中

```objc
    UNUserNotificationCenter* center = [UNUserNotificationCenter currentNotificationCenter];
    // 将通知请求 add 到 UNUserNotificationCenter
    [center addNotificationRequest:request withCompletionHandler:^(NSError * _Nullable error) {
        if (!error) {
            NSLog(@"推送已添加成功 %@", requestIdentifier);
            //你自己的需求例如下面：
            UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"本地通知" message:@"成功添加推送" preferredStyle:UIAlertControllerStyleAlert];
            UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:nil];
            [alert addAction:cancelAction];
            [[UIApplication sharedApplication].keyWindow.rootViewController presentViewController:alert animated:YES completion:nil];
            //此处省略一万行需求。。。。
        }
    }];
```

#### 远端推送

远端推送需要服务器按照苹果的格式推送 aps 内容：

```objc
{
  "aps" : {
    "alert" : {
      "title" : "iOS远程消息，我是主标题！-title",
      "subtitle" : "iOS远程消息，我是主标题！-Subtitle",
      "body" : "Dely,why am i so handsome -body"
    },
    "badge" : "2"
  }
}
```

## Notification Actions

这是 iOS10 推出的通知功能拓展，可加可不加，用来给推送添加一些按钮。

### 创建Action

#### button型：

```objc
 UNNotificationAction *lookAction = [UNNotificationAction actionWithIdentifier:@"action.join" title:@"接收邀请" options:UNNotificationActionOptionAuthenticationRequired];

UNNotificationAction *joinAction = [UNNotificationAction actionWithIdentifier:@"action.look" title:@"查看邀请" options:UNNotificationActionOptionForeground];

UNNotificationAction *cancelAction = [UNNotificationAction actionWithIdentifier:@"action.cancel" title:@"取消" options:UNNotificationActionOptionDestructive];
```

其中 `UNNotificationActionOptions` 是一个枚举类型，用来标识 Action 触发的行为方式分别为：

1. `UNNotificationActionOptionAuthenticationRequired = (1 << 0)`，需要解锁显示，点击不会进app
2. `UNNotificationActionOptionDestructive = (1 << 1)`，红色文字。点击不会进app
3. `UNNotificationActionOptionForeground = (1 << 2)`，黑色文字，点击会进app

#### 输入框型：

```objc
UNTextInputNotificationAction *inputAction = [UNTextInputNotificationAction actionWithIdentifier:@"action.input" title:@"输入" options:UNNotificationActionOptionForeground textInputButtonTitle:@"发送" textInputPlaceholder:@"tell me loudly"]
```

输入框型多了两个参数，`buttonTitle` 输入框右边的按钮标题，`placeholder` 输入框占位符。

### 创建category

#### button型

```objc
UNNotificationCategory *notificationCategory = [UNNotificationCategory categoryWithIdentifier:@"Dely_locationCategory" actions:@[lookAction, joinAction, cancelAction] intentIdentifiers:@[] options:UNNotificationCategoryOptionCustomDismissAction];
```

其中：

- `identifier`: 标识符是这个 category 的唯一标识，**用来区分多个 category,这个 id 不管是 Local Notification，还是 remote Notification，一定要有并且要保持一致**
- `actions`: 创建的 action 操作数组
- `intentIdentifiers`: 意图标识符 可在 `<Intents/INIntentIdentifiers.h>` 中查看，主要是针对电话、carplay 等开放的 API
- `options`: 一般都是 `UNNotificationCategoryOptionCustomDismissAction` 就行

> 不同通知，action 的方式也会不同。所以这个 `identifier` 是用来标识该用哪种方式处理的。对于 remote notification，需要在发给 APNS 的消息中添加 `category` 字段，值与 `identifier` 相同。对于 `local notification`，需要设置 `UNMutableNotificationContent` 实例的 `categoryIdentifier` 属性。

#### 输入框型

```objc
UNNotificationCategory *notificationCategory = [UNNotificationCategory categoryWithIdentifier:@"Dely_locationCategory" actions:@[inputAction] intentIdentifiers:@[] options:UNNotificationCategoryOptionCustomDismissAction];
```



### 将 category 添加到通知中心

```objc
// 将 category 添加到通知中心
 UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
 [center setNotificationCategories:[NSSet setWithObject:notificationCategory]];
```



### 事件的操作

上面代码并没有添加具体的操作方式，具体的操作都在前面说过的 `didReceiveNotificationResponse` 方法中进行：

```objc
//App通知的点击事件
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler{

    //点击或输入action
    NSString* actionIdentifierStr = response.actionIdentifier;
     //输入框型
    if ([response isKindOfClass:[UNTextInputNotificationResponse class]]) {

        NSString* userSayStr = [(UNTextInputNotificationResponse *)response userText];
        NSLog(@"actionid = %@\n  userSayStr = %@",actionIdentifierStr, userSayStr);
        //此处省略一万行需求代码。。。。
    }

     //button型
    if ([actionIdentifierStr isEqualToString:@"action.join"]) {

        //此处省略一万行需求代码
         NSLog(@"actionid = %@\n",actionIdentifierStr);
    }else if ([actionIdentifierStr isEqualToString:@"action.look"]){

        //此处省略一万行需求代码
        NSLog(@"actionid = %@\n",actionIdentifierStr);
 }
```

通过 `response` 拿到前面设置的 `identifier`，来按照需求，定制操作。

## Media Attachments

这个东西是给推送添加多媒体效果，比如图片，视频等。暂时用到的不多，用到再看。





[详细的参考文档](http://www.jianshu.com/p/c58f8322a278)

[推送拓展详解](http://www.jianshu.com/p/45933f5450a4)

[iOS10拓展完全解析](http://www.jianshu.com/p/2f3202b5e758)