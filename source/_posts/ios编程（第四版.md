title: ios编程(第四版) 学习笔记
date: 2016/7/31 14:07:12  
categories: iOS
tags: 
	- 学习笔记

---

本文是对《ios编程(第四版)》一书学习后摘录的笔记，属于ios开发基础。

<!--more-->
## 第一个ios应用
### 声明插座变量
```objc
@property(nonatomic, weak) IBOutlet UILabel *questionLabel
```
声明了一个叫questionLabel的插座变量.  
**IBOutlet**告诉Xcode需要使用Interface Builder关联该插座变量。

### 声明动作方法
```objc
- (IBAction)showQuestion:(id)sender{
	……
}
```
**IBAction**关键字告诉Xcode会使用Interface Builder关联该数据。之后创建关联。

### 应用图标
Images.xcassets目录下存放着所有图片，其中LaunchImage保存启动图片。

## objective-c
### 类方法
实例方法使用的字符是-，类方法使用字符+。  
类方法作用通常是创建对象，获取类的某些全局属性。

## 通过ARC管理内存
### copy
当某个属性是指向其他对象的指针，并且该对象的类有可修改的子类，如NSString、NSArray时，应该将该属性的内存管理特性设置为copy。相当于发送一个copy消息。  
当一个不可修改的对象进行copy时，返回原来对象。当一个可修改对象进行copy时，返回新创建的不可修改对象。

举个例子，比如有个`@property(nonatomic,copy) NSString *name`。如果不用`copy`而用`strong`那么将一个可修改的`NSMutableString`对象赋给`name`时，那就直接将`name`指向`NSMutableString`对象。此时，`NSMutableString`修改了自身的string值，那么相当于`name`的值也改变了，这和设计原则相悖。因此，需要使用`copy`,当`NSMutableString`对象赋给`name`时，深复制一个新的不可修改的对象，让`name`指向它，如果`NSString`对象赋给`name`时，浅赋值，将`NSString`的引用计数器加一。

copy和strong的区别就是：
- 对于不可修改对象没有区别，都是直接返回对象地址
- 对于可修改对象，copy新创建一个不可修改的原对象的实例，并返回地址。strong直接返回原对象地址。



## 视图与视图层次结构
### 视图层次结构
任何一个应用有且只有一个UIWindow对象。UIWindow包含应用中的所有视图。应用需要在启动时创建并设置UIWindow对象，然后为其添加其他视图。
层次结构中每个视图分别绘制自己。视图会将自己绘制到图层(layer)上，每个UIView对象都有一个layer属性，指向一个CALayer类的对象。

### 创建UIView子类
UIView子类模板会自动生成一个方法 **initWithFrame:**，该方法是UIView的指定初始化方法，带有一个CGRect结构类型的参数，该参数会被赋给UIView的frame属性。

```objc
@property （nonatomic） CGRect frame；
```

CGRect结构包含另外两个结构：origin和size。origin是CGPoint类型，包含两个float类型的x，y。size是CGSize结构，包含两个float类型的width和height。    
创建CGRect对象：CGRect firstFrame = CGRectMake（160，240，100，150）；
  
可以使用addSubView方法来添加子视图： [self.window addSubview:firstView]。  
每个UIView对象都还有一个superView属性，将一个视图作为子视图加入另一个视图后，会自动创建相应的反向关联。为避免强引用循环，superview是弱引用。  
bounds表示的矩形位于自己的坐标系，frame表示的矩形位于父视图的坐标系。

### 图形绘制
没看

## 视图：重绘与UIScrollView
### ScrollView
设置scrollView.contentSize = bigRect.size;这是整个滚动范围的大小。  
在scrollview中addSubView。

## 视图控制器
### 视图控制器
视图控制器是UIViewController类或其子类的对象。每个视图控制器都负责管理一个视图层次结构。
使用UITabBarController的类在两个视图控制器间切换。  

UIViewController有一个重要属性：

```objc 
@property (nonatomic, strong) UIView *view;
```

这个view就是视图的根视图。

### 创建视图层次结构
1. 覆盖UIViewController的loadView方法
```objc
-(void)loadView{
        BNRHypnosisView *backgroundView = [[BNRHypnosisView alloc] init];
        self.view = backgroundView;
}
```

	此时view尚未创建，不能对其进行addview操作。只能创建一个view让self.view指向。
2. 通过xib创建
.m中声明各个控件(注意控件使用弱引用)
```objc
@property(nonatomic, weak) IBOutlet UILabel *questionLabel
```

	再通过连线将File’s Owner的view以及各种控件连接到xib的图像上。

### 设置根视图控制器
UIWindow对象提供了setRootViewController：方法。当程序将某个视图控制器设置为UIWindow对象的rootViewController时，UIWindow对象会将该视图控制器的view作为子视图加入窗口。
```objc
BNRHypnosisViewController *hvc = [[BNRHypnosisViewController alloc] init];
self.window.rootViewController = hvc;
```

setRootViewController其实就是将ViewController的view设置为其subview。

### 加载nib文件
加载不同名的nib文件时，需要使用**initWithNibName:Bundle:**方法。该方法的两个参数，分别用于指定NIB文件文件名和其**所在的程序包**。如果是Bundle传入nil默认是[NSBundle mainBundle];

### UITabBarController
UITabBarController对象可以保存一组视图控制器。该对象会在屏幕底部显示一个标签栏，标签栏会有多个标签项，分别对应UITabBarController对象所保存的每一个视图控制器。单击某个标签项，UITabBarController对象就会显示该标签项所对应的视图控制器的视图。   

在APPDelegate中创建两个视图控制器，加入Tabbar的**viewControllers**属性中，并将tabbar设置为rootViewController
```objc
UITabBarController *tabBarController = [[UITabBarController alloc] init];
tabBarController.viewControllers = @[hvc,rvc] //两个viewController 可以写在tabbarController的viewDidLoad方法里
self.window.rootViewController = tabBarController;
```

设置标签项，使用**tabBarItem**属性：

```objc
    UIImage *orderImage = [UIImage imageNamed:@"tab_order"];
    UIImage *orderImageSel = [UIImage imageNamed:@"tab_order_active"];
    orderImage = [orderImage imageWithRenderingMode:UIImageRenderingModeAlwaysTemplate];
    orderImageSel = [orderImageSel imageWithRenderingMode:UIImageRenderingModeAlwaysTemplate];
    UITabBarItem *orderTabBarItem = [[UITabBarItem alloc] initWithTitle:@"订单" image:orderImage selectedImage:orderImageSel];
    orderNav.tabBarItem = orderTabBarItem;
```

为tabbar添加subview，UITabBarController里有一个**tabBar**的view

```objc
UIView *backview = [[UIView alloc] initWithFrame:self.tabBar.bounds];
[backview setBackgroundColor:[UIColor whiteColor]];
[self.tabBar addSubview:backview];
```


### 本地通知
本地通知需要创建一个UILocalNotification对象并设置其显示内容和提醒时间，然后调用UIApplication单例对象的scheduleLocalNotification:方法就实现了注册通知。
```objc
UILocalNotification *note = [][UILocalNotification alloc] init];
note.alertBody = @"xxx";
note.fireDate = date;
[[UIApplication sharedApplication] scheduleLocalNotification:note];
```

### 加载和显示视图
viewDidLoad方法检查视图控制器是否已经加载，每个UIViewController对象实现了viewDidLoad方法，该方法会在载入视图后被调用。为了实现视图延迟加载，在initWithNibName方法中不能访问任何view和view的任何子视图。凡是和view或view子视图有关的初始化代码，都应该在viewDidLoad方法中实现。  
另外一个方法viewWillAppear方法会在视图控制器的view添加到应用窗口之前被调用。
  
如果只要在应用启动后设置一次视图对象，那么用viewDidLoad，如果每次看到视图都需要对其进行设置，那么用viewWillAppear，类似于onCreate和onResume的区别。

### 与视图控制器及其视图交互
**application:didFinishLaunchingWithOptions:**	该方法中设置和初始化应用窗口的根视图控制器。该方法只会在应用启动后调用一次。如果从其他应用切换回本地应用，该方法不会被调用，如果关闭应用后台在重启才会再次调用。      
**initWithNibName：bundle：**该方法是UIViewController的指定初始化方法，创建视图控制器时，就会调用该方法。  
**loadView：**覆盖该方法，使用代码方式设置视图控制的的view属性。  
**viewDidLoad：**该方法会在视图控制器加载完视图后被调用。  
**viewWillAppear：**该方法会在视图控制器的view显示在屏幕上时被调用。  

## 委托与文本输入
### 委托（代理模式）
当某个特定事件发生时，发生事件的一方会向指定的目标对象发送一个之前设定好的动作消息。  
例：UITextField对象有一个委托属性，通过为UITextField对象设置委托，委托实现一个协议中的各个方法，UITextField对象会在发生事件时向委托发送相应消息，由委托处理该事件。  

类似于android中setOnClickListener(this); 继承onClickListener接口，这里则是设置delegate为self，遵守相应协议。不过android可以使用匿名内部类代替这里的this。

### 协议
凡是支持委托的对象，背后都有一个相应的协议，声明可以向该对象的委托对象发送的消息。委托对象需要更具这个协议选取需要的方法实现。如果一个类实现了某个协议中规定的方法，则称类遵守该协议。  

协议声明使用@protocol指令开头，后面跟协议的名称。最后使用@end结束  
使用@optional指令，可以将写在指令后的全部声明为可选的。  
发送方在发送可选方法前，会先向其委托发送另一个名为respondsToSelector的消息。所有oc对象都从NSObject继承 respondsToSelector：方法，该方法在运行时检查对象是否实现了指定的方法。  

声明示例：
```objc
@interface BNRHypnosisViewController()<UITextFieldDelegate>
@end
```
### 设置异常断点
当有问题无法找到错误原因时，可以通过添加异常断点定位有问题的代码。

### 类方法与实例方法
这里需要注意：
1. 类方法可以调用类方法。
2. 类方法不可以调用实例方法，但是类方法可以通过创建对象来访问实例方法。
3. 类方法不可以使用实例变量。类方法可以使用self，因为self不是实例变量。
 + 实例方法里面的self，是对象的首地址。
 + 类方法里面的self，是Class.
4. 类方法作为消息，可以被发送到类或者对象里面去（实际上，就是可以通过类或者对象调用类方法的意思）。

## UINavigationController
### UINavigationController对象
UINavigationController独享显示多个屏幕信息时，相应的UINavigationController对象会以栈的形式保存所有屏幕信息。初始化UINavigationController对象时，需要传入一个UIViewController对象作为UINavigationController对象的根视图控制器，且根视图控制器将永远位于栈底。

UINavigationController对象有一个名为**viewControllers**的属性，指向一个负责保存视图控制器的数组。**topViewController**属性是一个指针，指向当前位于栈顶的视图控制器。

UINavigationController是UIViewController的子类，所以UINavigationController有自己的视图。该视图有**两个子视图**：一个是**UINavigationBar对象**，一个是**topViewController的视图**。

初始化UINavigationController对象：
```objc
UINavigationController *navController = [[UINavigationController alloc]initWithRootViewController:viewController];
```

### 关联xib
不要将子视图设置在view的最顶端。在视图控制器中，view会衬于UINavigationBar下方，导致UInavigationBar遮挡view最顶端内容。（UITabBar同样）

在xib中按住Control拖拽到类拓展中，可以快速创建一个带IBOutlet前缀的属性。

设置xib文件时，要确保关联正确，产生错误的常见原因是：修改了某个插座变量的变量名，但是没有更新xib文件中的相应关联。

### 将视图控制器压入栈
使用UINavigationController对象时，通常会由处于栈顶的视图控制器来负责压入另一个视图控制器。
```objc
[self.navigationController pushViewController:detailController animated:YES];
```
视图控制器发送navigationController消息，可以得到指向UINavigationController对象的指针。

### 视图控制器间传递数据

### NavigationBar
UIViewController对象有一个navigationItem属性，类型为UINavigationItem。和NavigationBar不同，不是UIView的子类，不能显示，而是为UINavigationBar提供绘图所需内容。当UIViewController成为UINavigationController的栈顶对象时，UINavigationBar对象会访问该UIViewController对象的navigationItem，获取和界面显示相关的内容。

UINavigationItem除了设置title属性外，还可以设置其他属性：leftBarButtonItem，rightBarButtonItem和titleView。titleView可以将原有title的位置替换为一个view。

创建一个buttonItem：
```objc
- (void)initNavigationView{
    UIButton *backBtn = [[UIButton alloc] initWithFrame:CGRectMake(0, 0, 26, 44)];
    [backBtn setImage:[UIImage imageNamed:@"systemback"] forState:UIControlStateNormal];
    [backBtn addTarget:self action:@selector(backButtonPressedForOrder:) forControlEvents:UIControlEventTouchUpInside];
    UIBarButtonItem *leftItem  = [[UIBarButtonItem alloc]initWithCustomView:backBtn];
    [self.navigationItem setLeftBarButtonItem:leftItem];
}
```
