title: 一些 iOS 小技巧与注意点(持续更新)
date: 2017/2/9 10:07:12  
categories: iOS
tags:

 - 学习笔记
 - 持续更新

------

这篇中将收集一些关于 iOS 开发中的一些小技巧以及注意事项，可能比较零散。

<!--more-->

### 后台任务

app进入后台，会停止所有线程；需要在 `applicationDidEnterBackground` 中调用 `beginBackgroundTaskWithExpirationHandler` 申请更多的app执行时间，以便结束某些任务:

```objc
- (void)applicationDidEnterBackground:(UIApplication *)application {
    _backtaskIdentifier = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:^(void){
      
      	... 任何后台任务要做的事
          
        // 取消后台任务，在id不是UIBackgroundTaskInvalid的情况下，通过 endbackgroundTask 停止任务，并将id设置回invalid
        if (_backtaskIdentifier!=UIBackgroundTaskInvalid) {
            [[UIApplication sharedApplication] endBackgroundTask:_backtaskIdentifier];
            _backtaskIdentifier = UIBackgroundTaskInvalid;
        }
    }];  
}
```

其中 `_backtaskIdentifier` 是 `UIBackgroundTaskIdentifier` 类型

### 浏览器缓存策略

从浏览器的缓存中可以学到一些有关客户端缓存方案的实现方式

#### Cache-Control

Cache-Control 提供一个 max-age 字段，用于描述过期时间。浏览器是否再次请求数据决定于:

```
浏览器当前时间 > 浏览器上次请求时间 + max-age
```

#### Last-Modified/If-Modified-Since

浏览器每次请求后，服务器会返回一个资源在服务端上次更新的时间。下次请求的时候会带上这个时间。如果服务端发现这个时间之后没有更新过资源就会返回 304，让浏览器用本地数据

但是如果碰到某些文件会被定期生成, 而内容其实并没有发生任何变化的情况，这种情况下 Last-Modified改变了, 但这种情况其实应该返回304。因此设计了 ETag 方式。

ETag 用于唯一的标识一个文件。如果两个文件 ETag 相同，那么就表示两个文件是一致的，不会重复返回。

> 这两种方式：
>
> - 一种是用于客户端是否要去请求。
> - 一种是用于客户端请求了服务端是否要返回数据。

### atomic 与线程安全

`atomic`的原子粒度是Getter/Setter，但对多行代码的操作不能保证原子性：

```objc
@property (atomic, assign) int num;

// thread A
for (int i = 0; i < 10000; i++) {
    self.num = self.num + 1;
    NSLog(@"Thread A: %d\d ",self.num);
}

// thread B
for (int i = 0; i < 10000; i++) {
    self.num = self.num + 1;
    NSLog(@"Thread B: %d\d ",self.num);
}
```

最终的结果不一定是 20000，因为这个过程包含了，读取 num，给 num 加一，写 num 三个操作。读写是原子性的，但是这三个操作不是原子性的。

### `__attribute__` 的使用

`__attribute__` 是编译指令，一般以`__attribute__(xxx)`的形式出现在代码中，方便开发者向编译器表达某种要求。

#### constructor

`__attribute__((constructor))` 加上这个属性的函数会在可执行文件 load 时被调用。可以理解为在 main() 函数调用前执行：

```c++
__attribute__((constructor))
static void beforeMain(void) {
    NSLog(@"beforeMain");
}
__attribute__((destructor))
static void afterMain(void) {
    NSLog(@"afterMain");
}
int main(int argc, const char * argv[]) {
    NSLog(@"main");
    return 0;
}

// Console:
// "beforeMain" -> "main" -> “afterMain"
```

constructor 和 +load 都是在 main 函数执行前调用，但 +load 比 constructor 更加早一丢丢，因为 dyld（动态链接器，程序的最初起点）在加载 image（可以理解成 Mach-O 文件）时会先通知 objc runtime 去加载其中所有的类，每加载一个类时，它的 +load 随之调用，全部加载完成后，dyld 才会调用这个 image 中所有的 constructor 方法。所以 constructor 是一个干坏事的绝佳时机：

1. 所有Class都已经加载完成
2. main 函数还未执行
3. 无需像 +load 还得挂载在一个Class中

#### section

通常情况下，编译器会将对象放置于DATA段的*data*或者*bss*节中。但是，有时我们需要将数据放置于特殊的自定义 section 中，此时*section*可以达到目的。例如，BeeHive中就把module注册数据存在__DATA数据段里面的“BeehiveMods”section中。

```objc
char * kShopModule_mod __attribute((used, section("__DATA,""BeehiveMods"""))) = """ShopModule""";
```

其中 `used` 告诉编译器，即使它没有被使用，也不要优化掉。

### nil Null Nil NSNull 区别

nil 定义某实例对象为空：

```objc
NSObject* obj = nil;
```

Nil 定义类为空

```objc
Class someClass = Nil;
```

NULL 用于 c 语言数据类型的指针为空：

```objc
char *pointerToChar = NULL;
```

NSNull 是空对象，放在 NSArray 中

```objc
NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionary];
mutableDictionary[@"someKey"] = [NSNull null];
```

### SafeArea

iOS11 提供了 `safeAreaLayoutGuide` 和 `safeAreaInsets` 来帮助刘海屏的适配。

前者你可以把它当做是一个 view。我们一般配合 Masonry 使用。Masonry 提供了 `mas_safeAreaLayoutGuide`，`mas_safeAreaLayoutGuideRight/Left/Top/Bottom` 来辅助布局： 

```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
     make.edges.equalTo(self.view.mas_safeAreaLayoutGuide).inset(10.0);
}];
......
[rightTopView mas_makeConstraints:^(MASConstraintMaker *make) {
     make.right.equalTo(self.view.mas_safeAreaLayoutGuideRight);
     make.top.equalTo(self.view.mas_safeAreaLayoutGuideTop);
     make.width.height.equalTo(@(size));
}];
```

后者则是一个 inset，适合在 frame 布局的时候使用。但是要注意，`safeAreaInsets` 是在系统方法 `viewSafeAreaInsetsDidChange` 之后才会变为正确的值的，所以我们需要在之后的生命周期方法，比如 `viewDidLayoutSubviews` 方法中设置：

```objc
static inline UIEdgeInsets sgm_safeAreaInset(UIView *view) {
    if (@available(iOS 11.0, *)) {
        return view.safeAreaInsets;
    }
    return UIEdgeInsetsZero;
}

- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    UIEdgeInsets safeAreaInsets = sgm_safeAreaInset(self.view);
    CGFloat height = 44.0; // 导航栏原本的高度，通常是44.0
    height += safeAreaInsets.top > 0 ? safeAreaInsets.top : 20.0; // 20.0是statusbar的高度，这里假设statusbar不消失
    if (_navigationbar && _navigationbar.height != height) {
        _navigationbar.height = height;
}
```



### UIView 的生命周期

```objc
- (void)didAddSubview:(UIView *)subview;
- (void)willRemoveSubview:(UIView *)subview;
- (void)willMoveToSuperview:(nullable UIView *)newSuperview;
- (void)didMoveToSuperview;
```

 `willMoveToSuperview` 会在添加到父视图和从父视图移除的时候调用，移除的时候参数为 `nil`（类似于 `willMoveToParentViewController`）

 `didMoveToSuperview` 也比较重要，在这个方法中可以拿到父视图的大小，进行相应的布局

### 判断是否点击了某个 View

```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [touches anyObject];
    // 将 touch 转化到 View 的坐标系
    CGPoint touchPoint = [touch locationInView:self];
    if (CGRectContainsPoint(self.bounds, touchPoint)) {
        // 点在了 View 中
    }
}
```



### 画一条一个像素的线

根据当前屏幕的缩放因子计算出1 像素线对应的Point，然后设置偏移

```objc
#define SINGLE_LINE_WIDTH           (1 / [UIScreen mainScreen].scale)
#define SINGLE_LINE_ADJUST_OFFSET   ((1 / [UIScreen mainScreen].scale) / 2)

UIView *view = [[UIView alloc] initWithFrame:CGrect(x - SINGLE_LINE_ADJUST_OFFSET, 0, SINGLE_LINE_WIDTH, 100)];
```

iOS设备上，有逻辑像素（point）和 物理像素（pixel)之分。point和pixel的比例是通过[[UIScreen mainScreen] scale]来制定的。在没有视网膜屏之前，1point = 1pixel；但是2x和3x的视网膜屏出来之后，1point等于2pixel或3pixel。

在UI设计师提供的设计稿标注，和在代码中设置frame，其中x,y,width,height的单位是 **逻辑像素（point）**；GPU在渲染图形之前，系统会将**逻辑像素（point）**换算成 **物理像素（pixel）**。

### 缓存行高

在cell离开屏幕后，保存行高：

```objc

- (void)tableView:(UITableView *)tableView didEndDisplayingCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath 
{
  NSString *key = [NSString stringWithFormat:@"%ld",    (long)indexPath.row];
  [self.heightDict setObject:@(cell.height) forKey:key];
  DEBUG_LOG(@"第%@行的计算的最终高度是%f",key,cell.height);
}
```

如果已经缓存了行高，那么直接返回高度，否则返回预估行高：

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    NSString *key = [NSString stringWithFormat:@"%ld",indexPath.row];
    if (self.heightDict[key] != nil) {
       NSNumber *value = _heightDict[key];
       DEBUG_LOG(@"%@行的缓存下来的高度是%f",key,value.floatValue);
       return value.floatValue;
    }
    return UITableViewAutomaticDimension;
}
```



### 部分页面支持旋转

在一些视频应用中，播放视频的时候是支持屏幕旋转的，但是其他时候则不支持，iOS 在 UIViewController 中提供了一个方法 `supportedInterfaceOrientations()`。

当 iOS 设备旋转到一个新的方向时，`supportedInterfaceOrientations()` 方法就会被调用。如果这个方法的返回值中包含新的方向，应用程序就会旋转当前视图，否则不旋转。每个视图控制器都可以重写这个方法，对于单个试图控制器，可以在特定条件下支持特定的方向。

```swift
override func supportedInterfaceOrientations)_ -> UIInterfaceOrientationMasl {
    return UIInterfaceOrientationMask(rawValue:(UIInterfaceOrientationMask.portait.rawValue | UIInterfaceOrientationMask.landscapeLeft.rawValue))
}
```



### 通过 GCD 判断是否是主队列

先要明白队列和线程的关系。主线程中除了主队列还有可能运行其他的全局队列。而主队列只会在主线程中运行

关于如何判断是否是主队列，可以使用 GCD：

```objc
BOOL RCTIsMainQueue()
{
  static void *mainQueueKey = &mainQueueKey;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    dispatch_queue_set_specific(dispatch_get_main_queue(),
                                mainQueueKey, mainQueueKey, NULL);
  });
  return dispatch_get_specific(mainQueueKey) == mainQueueKey;
}
```

其实就是在主线程中添加了一个 key-value 映射，只有在主线程的情况下，才能拿到这个 key-value

还有一种方式，通过获取当前队列的 label 和 主队列的 label 进行比较：

```objc
dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL) == dispatch_queue_get_label(dispatch_get_main_queue())
```



### 模拟器弹出键盘

一般情况下，模拟器的输入框是不会弹出键盘的，我们可以设置其为弹出键盘：`command+shift+k`。其实勾掉了选项中的 connect kardware keyboard：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/keyboard_1.png?raw=true)

### import 尖括号和双引号的区别

import "xxx.h"的路径搜索顺序：

1. USE_HEADERMAP
2. USER_HEADER_SEARCH_PATHS
3. HEADER_SEARCH_PATHS

import <xxx.h> 的路径搜索顺序：

1. 系统路径
2. HEADER_SEARCH_PATHS

USE_HEADERMAP 是在 Build Settings 中的 Use Header Maps 中设置，默认为 YES，表示系统会自动为编译期提供一个文件映射。我们自己 new 出来的文件就是通过它找到的。

其实就是双引号是自己创建的文件里找，尖括号是从系统的文件里找。没找到的话，都会到 HEADER_SEARCH_PATHS 中找。

### Framework Search Paths 和 Header Search Paths

一般情况下，直接把文件或者 framework 拖进工程里，并且勾上 `copy if needed`，不会出现任何问题。但是有时候，我们的头文件或者 framework 并不能直接放到工程里。比如一个 RN 应用，framework 是保存在 RN 的 src 中的某个文件夹内的。这个时候就会出现头文件找不到的问题。

因为通过 `copy if needed` 添加的文件会被 Xcode 自动添加到 Framework Search Paths 或者 Header Search Paths 里。但是如果不是这种情况，我们就需要手动添加 Search Paths

> 主工程中引入了目标库，子target没有引入目标库。由于目标库已经在工程中了，因此只要在子target只要设置头文件搜寻地址即可。
>
> 以上适用于静态库，对于动态库，主工程中需要引入目标库，子 target 中还需要将动态库通过 Link Binary With Libraries 引入，不用设置头文件搜寻地址了。

### 为什么不在 iOS 项目中使用 redux

redux 将所有状态组合在一起，配合 react 食用更佳，那么我们为什么不把他使用子啊 iOS 中呢？我想到了两个理由：

1. oc 或者 swift 是强类型语言。如果把状态都放在 store 里维护，意味着 store 里需要 import 各种 model 类。产生强耦合。而 js 则是弱类型。不用在意 store 里的某个属性究竟是什么类型，也就不需要 import 那么多 model。
2. react 是相应式的，store 里的属性修改了就能反映到 UI 上，但是 iOS 编程不是响应式的，即使 store 里的实行改变了，你还是要使用 `setText` 等类似方法，刷新 UI。所以即使使用也最好配合 RxSwift

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_ios.png?raw=true)



### 封装原则

- 明确目标：首先靠考虑好，封装要做什么，不要写着写着换目标。
- 单一功能原则：其次，要将一个大目标分解成若干步小目标分别实现。然后串联起来
- 外界知道最少：找出外界必须知道的，作为封装输出
- 重复代码复用：找出重复代码，塞到封装内
- 写伪代码：逻辑复杂，不要想着一步到位。先写伪代码，然后把相同的部分归类。

### Image.Assets 存放图片和在文件夹里存放的区别

Assets.xcassets 一般是以蓝色文件夹形式在工程中，以Image Set的形式管理。当一组图片放入的时候同时会生成描述文件Contents.json。**且在打包后以Assets.car的形式存在，以此方式放入的图片并不在mainBundle中**，不能使用 contentOfFile 这样的API来加载图片（因为这个方法相当于是去mainBundle里面找图片，但是这些图片都被打包进了Assets.car文件），interface builder中使用图片时不需要后缀和倍数标识（@2x这样的）优点是性能好，节省Disk。

直接拖到文件夹里的图片，都是存在 mainBundle 中的。所以可以使用 contentOfFile 来加载图片。另外我们也可以把资源文件打成一个 bundle，放在主工程下（mainBundle 下）。 

### copy groups 和 copy folder reference 的区别

#### 什么是 Group

Group 其实是仅在 Xcode 中虚拟的用来组织文件的一种方式, **它对文件系统没有任何影响**, 无论你创建或者删除一个 Group, 都不会导致实际的文件的增加或者移除。

#### 两者区别

打包的时候， xode 目录下的所有文件，除了 Image.Assets 以及动态库，都会保存到 mainBundle 中去（静态库是在 mainBundle 中的）。以 `copy groups` 引入的文件，没有文件层级，就**相当于直接把 group 里的文件扔到了 mainBundle 中去**，而以 `copy folder reference` 引入的文件会保持文件层级。

Group 在我们的工程中就是黄色的文件夹, 而 Folder 是蓝色的文件夹。





### 基本类型的指针

我们知道，基本类型的变量作为入参传入函数产生改变时不会影响外部的变量值。所以如果不能返回的话，就需要传入一个基本类型的指针：

```objc
BOOL roolback = YES;
BOOL *r = &roolback;
---
// 假设传入了某个方法，然后在其中改变了值
*r = NO;
---
NSLog(@"%d，roolback 从1变为了0",roolback);   
```

非基本类型的指针的赋值直接就是 `p = Person()` 这种地址的赋值就行了，**不存在 `*p` 的情况**。基本类型的指针必须通过 `*r` 拿到堆上的值 `*r = No。

### 为文本加上下划线

基本就是用 `NSMutableAttributedString` 代替原来的 `NSString`，然后为文本长度的 range 内添加一个下划线的属性。

下面演示一个为按钮添加下划线:

```objc
NSMutableAttributedString *string = [[NSMutableAttributedString alloc] initWithString:button.titleLabel.text];
NSRange strRange = {0,[string length]};
[string addAttribute:NSUnderlineStyleAttributeName value:[NSNumber numberWithInteger:NSUnderlineStyleSingle] range:strRange];
[button setAttributedTitle:string forState:UIControlStateNormal];
```



### 如何让 xib 中的两个视图平分父视图

说是平分，其实就是两个视图的宽度相同。最开始的办法是设置一个空白视图居于父视图的中间，然后两个视图分别贴着这个空白视图的左右，但是这个方法非常的蠢。其实只要选中两个视图，然后设置为 `equal width` 就行了，就在设置约束的地方。

### VC 中使用 self.view 获取屏幕宽高出错

**错误描述：**在 `viewDidLoad` 中视图通过 `self.view` 获取屏幕宽高，但是获取数值有误。

**原因：**这种情况一般是使用 xib 设置页面的时候出现。使用xib设置页面的时候，选择的是iPhone7的布局，宽度是375.当从 xib 加载页面后，`viewDidLoad` 时，约束没有更新。所以 `self.view` 的宽度和高度还是 xib 设计的高宽。要等到 `viewDidAppear` 时页面约束就会更新。

建议不要在 `viewDidLoad` 中直接使用 `self.view.bounds` 的高宽来计算其他属性的 `frame`。 而是使用 `[UIScreen mainScreen].bounds` 获取手机屏幕的真实高宽。

### 如何在二级页面隐藏 `tabbar`
一般会在 `TabBarController` 中设置多个 `NavigationController` 作为各个 `tab` 的 `ViewController`。我们只要写一个 `BaseViewController`，在其中重写 `pushViewController:animated` 方法，在其中判断是否需要隐藏就行。设置 `hidesBottomBarWhenPushed` 属性来控制在跳转的时候隐藏。
```objc
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    if (self.viewControllers.count) {
        viewController.hidesBottomBarWhenPushed = YES;
    }
    [super pushViewController:viewController animated:animated];
}
```


### 如何在 `pop` 的时候清理资源
仍然是在 `BaseViewController` 中重写 `popViewControllerAnimated:` 方法。在其中调用特定方法，来做一些清理资源的善后工作。
```objc
- (UIViewController *)popViewControllerAnimated:(BOOL)animated{
    if ([[self.viewControllers lastObject] isKindOfClass:[BaseViewController class]]) {
        BaseViewController *viewController=[self.viewControllers lastObject];
        [viewController willPopViewController];
    }
    return [super popViewControllerAnimated:animated];
}
```


### `UIImage` 的渲染模式 `imageWithRenderingMode:`
该方法用来设置属性 `renderingMode`。它使用 `UIImageRenderingMode` 枚举值来设置图片的 `renderingMode` 属性。该枚举中包含下列值：
```objc
UIImageRenderingModeAutomatic  // 根据图片的使用环境和所处的绘图上下文自动调整渲染模式。
UIImageRenderingModeAlwaysOriginal   // 始终绘制图片原始状态，不使用Tint Color。  
UIImageRenderingModeAlwaysTemplate   // 始终根据Tint Color绘制图片，忽略图片的颜色信息。
```

应用场景是什么呢？比如设置一个 `tabbar` 的 `UIImage` 如果不将其设置为 `UIImageRenderingModeAlwaysOriginal`。那么图片的选中状态的颜色就会跟随系统的 `tintColor`。


### 封装一个没有数据时显示空白页
没有数据的时候通常需要显示一个空白页。空白页长得都差不多，基本上都是一段话加上一张图。我们当然不要每次都写一遍这个视图，因此就需要将这些操作放到 base 中去。

在 `BaseViewController` 中添加一个 `reloadDataWithBlank` 方法，每次从网络获取完数据后，调用即可（省去了每次都要判断数据是否为空的麻烦）：
```objc
- (void)reloadDataWithBlank{
		//多段就看每个数组，一段就直接看元素个数
    if (_isMultiSection) {
        BOOL isCountZero=YES;
        for (NSArray *array in self.dataSource) {
            if (array.count!=0) {
                isCountZero=NO;
                break;
            }
        }
        if (isCountZero) {
            [_tableView addSubview:self.blankHintView];
        }else{
            [self.blankHintView removeFromSuperview];
        }
    }else{
        if (self.dataSource.count==0) {
            [_tableView addSubview:self.blankHintView];
        }else{
            [self.blankHintView removeFromSuperview];
        }
    }
    [_tableView reloadData];
}
```
其中 `blankHintView` 就是空白提示视图：

```objective-c
-(UIView *)blankHintView{
    if (!_blankHintView) {
        [self initBlankHintView];
    }
    return _blankHintView;
}

-(void)initBlankHintView{
    _blankHintView=[[UIView alloc]initWithFrame:CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height)];
    UIImageView *imageView=[[UIImageView alloc]initWithFrame:CGRectMake(self.tableView.width/2-40,self.tableView.height/2-40-30,80,80)];
    if (_blankHintImage) {
        imageView.image = _blankHintImage;
    }else{
        imageView.image = [UIImage imageNamed:@"nonetworkdefault"];
    }
    imageView.contentMode=UIViewContentModeScaleAspectFit;
    UILabel *hintLabel = [[UILabel alloc]initWithFrame:CGRectMake(imageView.x-10, imageView.y+imageView.height, 200, 30)];
    hintLabel.numberOfLines = 0;
    hintLabel.lineBreakMode = NSLineBreakByWordWrapping;
    hintLabel.textColor=[UIColor colorWithHexString:@"9e9e9e"];
    hintLabel.backgroundColor = [UIColor clearColor];
    hintLabel.textAlignment=NSTextAlignmentLeft;
    hintLabel.font=FONT(15);
    hintLabel.text=_blankHintString;
    [_blankHintView addSubview:hintLabel];
    [_blankHintView addSubview:imageView];
    [hintLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(imageView.mas_bottom).offset(0);
        make.centerX.equalTo(imageView.mas_centerX).offset(0);
        make.height.equalTo(@30);
    }];
}
```



每次在 `viewDidLoad` 中，或者在这个方法内再创建一个功能更明确的 `loadDefaultDataSource` 方法，在其中中添加默认的 `title` 和 `image` 即可。

```objective-c
-(void)loadDefaultDataSource{
    [super loadDefaultDataSource];
    self.blankHintImage = [UIImage imageNamed:@"default_nomessage"];
    self.blankHintString=@"暂无通知消息";
}
```

### 设置手势事件
关于手势，通常需要先声明一个手势，再在另一处添加这个手势的处理方法，这样设置事件和方法本身就会分隔开来，回顾代码的时候找起来会很麻烦。可以为 `UIView` 设置添加一个处理手势的范畴(category)，在其中通过 block 回调的方式将方法保存起来：
```objc
@implementation UIView (BlockGesture)
  
- (void)addTapActionWithBlock:(GestureActionBlock)block{
    UITapGestureRecognizer *gesture = [self associatedValueForKey:_cmd];
    if (!gesture){
        gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleActionForTapGesture:)];
        [self addGestureRecognizer:gesture];
        [self setAssociateValue:gesture withKey:_cmd];
    }
    [self setAssociateCopyValue:block withKey:&kActionHandlerTapBlockKey];
    self.userInteractionEnabled = YES;
}

- (void)handleActionForTapGesture:(UITapGestureRecognizer*)gesture{
    if (gesture.state == UIGestureRecognizerStateRecognized){
        GestureActionBlock block = [self associatedValueForKey:&kActionHandlerTapBlockKey];
        if (block){
            block(gesture);
        }
    }
}
```
先把 `gesture` 保存起来，再把手势对应的 `block` 保存起来。使用的时候调用即可：
```objc
[label addTapActionWithBlock:^(UIGestureRecognizer *gestureRecoginzer) {
	...
}];
```



### 图像显示过程与一些注意事项

#### 显示过程

**这里首先强调，这里说的所有都是针对 View 而不是 Controller。** 自动布局将视图显示到屏幕上的步骤总共分为三步：

1. 更新约束。它为布局准备好必要的信息，而这些布局将在实际设置视图的 frame 时被传递过去并被使用。你可以通过调用  `setNeedsUpdateConstraints` 来触发这个操作。谈到自定义视图，**可以重写 `updateConstraints` 来为你的视图的本地约束进行修改（一般不会用啦）**，但要确保在你的实现中修改了任何你需要布局子视图的约束条件**之后**，调用一下 `[super updateConstraints]`。
2. 布局。将约束条件应用到视图上。可以**通过重写 `layoutSubViews` 实现**，也就是在调用 `[super layoutSubviews]` 前修改一些约束信息。可以通过调用 `setNeedsLayout` 来触发一个操作请求，这并不会立刻应用布局，而是在稍后再进行处理。因为所有的布局请求将会被合并到一个布局操作中去。可以调用  `layoutIfNeeded` 来强制系统立即更新视图树的布局。
3. 显示。可以通过调用 `setNeedsDisplay` 来触发，这将会导致所有的调用都被合并到一起推迟重绘。重写熟悉的 `drawRect:` ，通过这个方法我们可以在 `UIView` 中绘制图形，比如绘制个角标之类的。

![图像显示流程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/display流程.png?raw=true)



#### `layoutSubViews` 的注意事项

`layoutSubViews` 方法给 View 提供了一个统一设置子视图大小布局的地方，不能在 `init` 方法中。因为当外部创建该 view 时，如果外部调用 `init` 方法而不是调用 `initWithFrame` (即没有给这个 view 设置大小），那么 view 中子视图就无法正确初始化出大小。因此，**子视图的 `bounds` 等属性（例如 `center` 等）的设置必须放在 `layoutSubviews` 中，即外部 view 大小已经确定，就要显示的时候。**

---

**记一个错误容易犯的错误：**

`frame`，`center` 都是相对于父 View 的。所以如果你要让一个子 View 在 父 View 居中，你绝对不能写 `subView.center = parentView.center`，因为 `parentView.center` 又是相对于其父 View 的。你应该通过父 View 的 `bounds` 计算得到:

```objc
- (void)layoutSubViews {
  subView.center = CGPointMake(self.bounds.size.width/2,self.bounds.size.height/2);
}
```

**记另一个容易犯的错误**

上面重写了 `layoutSubViews` 这个方法，并没有调用父类方法 `[super layoutSubViews]`，那么到底要不要调用呢？答案是最好调用，否则可能会出错。下面简述可能会出错的情况。

比如一个 CollectionViewCell，其中有一层图层 `contentView`。我们添加的视图一般都是添加在 `contentView` 上。当 cell 的 bounds 发生变化的时候，`contentView` 的大小不会立即变化，而是在调用了系统的 `[super layoutSubViews]` 后才会将 `contentView` 变为和 cell 一样大。如果你不调用父类方法，那么你就不能将你添加的子视图的位置依赖于 `contentView` 的 bounds

其实就是系统会在 `[super layoutSubViews]` 中修改系统自身添加的子视图的位置信息。如果你要用到系统的子视图，你就必须要先调用 `[super layoutSubViews]`

---



**注意，不是添加子视图。子视图在 `init` 方法中添加。一定不要在 `layoutSubviews` 方法中添加子视图。因为 `layoutSubviews` 方法会被多次调用，不可能每次调用都添加一遍子视图吧。**

**另外最好不要孤立地改动在 `layoutSubviews` 中设置过的子视图（比如大小，位置等）。因为 `layoutSubviews` 会被多次调用。在被调用后，改动又会变回去了。要改动也是要先将要改动的属性保存起来，让 `layoutSuibviews` 调用的时候通过这个属性设置 View。**

> 这个方法只是在要设置子视图 frame 的时候重写。如果你使用的是 autolayout 来设定位置，那么直接在 `init` 中设置 constraints 就可以了，因为这个方法会被调用多次，在这个方法里添加约束会导致约束重复添加。
>
> 什么时候用 autolayout 什么时候用 frame 就见仁见智了。一般来说只与父视图有关的话，那么就用 frame 设置位置，如果与兄弟视图有关的话，还是用 autolayout 教好一些。
>
> 同样的还有 ViewController 的 `viewDidLayoutSubviews` 方法，这个方法会在 VC 的 view 调用其 `layoutSubViews` 后调用（相当于调用了 `[super layoutSubViews]`）。如果要设置 VC 中的视图的 frame，可以考虑在这个方法里设置。但是一般我们都是在 `viewDidLoad` 方法里设置的无论是 frame 还是 constraint。

#### frame 修改 AutoLayout

AutoLayout 最后也是将约束设置为要展示的位置信息。所以我们也是可以使用 frame 来改变 autolayout 产生的位置的。但是修改的时机很重要。比如你在 xib 中设置了约束，然后你再 `viewDidLoad` 方法中修改了某个空间的 frame 那是没有用的。**因为，autolayout 的布局，本质上还是 frame 的形式。系统会在 layout subViews 时加载约束信息，将其转换为 frame。**所以如果你想改变 autolayout 的布局，你需要在 `viewDidLayoutSubviews` 中设置 frame

#### 改变约束的注意事项

**任何地方都能够添加修改约束**（包括 init 方法）。但是修改过约束后，并不能触发视图的更新，所以一般要调用 `layoutIfNeeded` 方法查看更新约束后的视图。更新视图的动画方法一般设置为：

```objc
[UIView animateWithDuration:1.0f delay:0.0f usingSpringWithDamping:0.5f initialSpringVelocity:1 options:UIViewAnimationOptionCurveEaseInOut animations:^{
        [self.view layoutIfNeeded];
} completion:NULL];
```

#### `updateConstraints` 的使用时机

不要将 `updateConstraints` 用于视图的初始化设置。**当你需要在单个布局流程（single layout pass）中添加、修改或删除大量约束的时候，用它来获得最佳性能。如果没有性能问题，直接更新约束更简单。**

如果一定要用这个方法，那么在更新约束之前，请先移除之前设置的约束，否则约束重复了会编译不过的。 移除约束的方法是：` [self removeConstraints:self.constraints];`。

**对于手撸一个 view 来说，一般不用重写 `updateConstraints`，只有在某些情况要大量修改约束的时候才用到，请把建立 constraints 的工作放在 `init` 或 `viewDidLoad` 階段。**

#### 关于 `drawRect`

`drawRect` 方法通过 `CGContextRef context = UIGraphicsGetCurrentContext();` 拿到上下文句柄，然后将自己要画的图形添加到这个上下文上。具体可以在要用的时候看看例子。

### 设置 view 的一些注意事项

- 重写 `UIView` 的 `initWithFrame:` 而不是 `init` 方法。 因为当外部调用 `init` 的方法的时候，其内部也会默默地调用 `initWithFrame：`方法。
- 不要在构造方法里面直接取自身(self,或者说本视图)的宽高,这时候取到的宽高是不准的.


- 当在代码中设置视图和它们的约束条件时候，一定要记得将该视图的 `translatesAutoResizingMaskIntoConstraints` 属性设置为 NO。如果忘记设置这个属性几乎肯定会导致不可满足的约束条件错误。
- 在 `initWithFrame:` 方法中将子控件加到 view 而不是设置尺寸。因为 view 有可能是通过 `init` 方法创建的，这个时候 view 的 frame 可能是 不确定的。这种情况下各个子控件的尺寸都会是0，因为这个 view 的 frame 还没有设置。设置尺寸的工作放在 `layoutSubviews` 中去做。
- `allowsGroupOpacity` 属性允许子控件的不透明度继承于其父控件，默认是开启的 `yes`。不过这会影响性能，自定义控件的时候最好设置为 `self.layer.allowsGroupOpacity = NO;`
- `clipsToBounds` 是 `UIView` 的属性，如果设置为 `yes`，则不显示超出父 View 的部分；`masksToBounds` 是 `CALayer` 的属性，如果设置为 `yes`，则不显示超出父 View layer 的部分.
- 设置视图的时候一定要先设置大小再设置 center ，center 是为了确定 CGRect 的，如果当时CGRect为0，那么此时设置 center，就像当于给 CGRect 设置了 origin。
- 


### 用 UIImageView 播放动图

app 中的加载等候经常需要播放一个动图，那么怎么让图片动起来呢？`UIImageView` 就能解决。

把所需要的 GIF 打包到一个叫做 Loading 的 Bundle 中去，加载 Bundle 中的图片：

```objc
- (NSArray *)animationImages
{
  	// 拿到所有文件的文件名的数组
    NSFileManager *fielM = [NSFileManager defaultManager];
    NSString *path = [[NSBundle mainBundle] pathForResource:@"Loading" ofType:@"bundle"];
    NSArray *arrays = [fielM contentsOfDirectoryAtPath:path error:nil];
	
  	// 遍历文件名，拿到文件名对应的 UIImage 加入到 UIImage 的数组中
    NSMutableArray *imagesArr = [NSMutableArray array];
    for (NSString *name in arrays) {
        UIImage *image = [UIImage imageNamed:[(@"Loading.bundle") stringByAppendingPathComponent:name]];
        if (image) {
            [imagesArr addObject:image];
        }
    }
    return imagesArr;
}

- (void)viewDidLoad {
    [super viewDidLoad];

    UIImageView *gifImageView = [[UIImageView alloc] initWithFrame:frame];
  	// 将 UIImage 数组设置给 UIImageView
    gifImageView.animationImages = [self animationImages]; //获取Gif图片列表
    gifImageView.animationDuration = 5;     //执行一次完整动画所需的时长
    gifImageView.animationRepeatCount = 1;  //动画重复次数
    [gifImageView startAnimating];
    [self.view addSubview:gifImageView];
}
```



### 获取一个指定的 View

想要获取一个指定的 view 的方法是遍历一个 view 的所有 subview，然后判断 view 的类型是否是指定的。在 `MBProgressHUD` 中的方式是：

```objc
+ (MBProgressHUD *)HUDForView:(UIView *)view {
    NSEnumerator *subviewsEnum = [view.subviews reverseObjectEnumerator];
    for (UIView *subview in subviewsEnum) {
        if ([subview isKindOfClass:self]) {
            return (MBProgressHUD *)subview;
        }
    }
    return nil;
}
```

这个方法是找到并隐藏相应 hud。这里面使用了 `NSEnumerator` 这个枚举类，通过 `reverseObjectEnumerator` 反向浏览集合。

### 获取当前 ViewController

### 通过 View

为了做到数据与视图的分离，我们一般会将一个页面的局部视图以自定义 `UIView` 的方式独立出来，如果在该视图中有触发事件(事件处理不需要父视图的上下文)，就会遇到在 `UIView` 中获取 `UIViewController` 的情况，可以写一个 `UIView` 的范畴 `UIView(UIViewController)`：

```objc
#pragma mark - 获取当前view的viewcontroller
+ (UIViewController *)getCurrentViewController:(UIView *) currentView
 {
       for (UIView* next = [currentView superview]; next; next = next.superview)
      {
             UIResponder *nextResponder = [next nextResponder];
             if ([nextResponder isKindOfClass:[UIViewController class]])
             {
                  return (UIViewController *)nextResponder;
             }
      }
     return nil;
}
```

### 通过 rootViewController

```objc
- (UIViewController *)findCurrentViewController
{
    UIWindow *window = [[UIApplication sharedApplication].delegate window];
    UIViewController *topViewController = [window rootViewController];
    
    while (true) {
        
        if (topViewController.presentedViewController) {
            
            topViewController = topViewController.presentedViewController;
            
        } else if ([topViewController isKindOfClass:[UINavigationController class]] && [(UINavigationController*)topViewController topViewController]) {
            
            topViewController = [(UINavigationController *)topViewController topViewController];
            
        } else if ([topViewController isKindOfClass:[UITabBarController class]]) {
            
            UITabBarController *tab = (UITabBarController *)topViewController;
            topViewController = tab.selectedViewController;
            
        } else {
            break;
        }
    }
    return topViewController;
}
```



### pop 到指定 ViewController

`UINavigationController` 有个 Property，是一个存储所有 push 进 navigationcontroller 的视图的集合，是一个栈结构，当我们要 POP 到某个  ViewController 的时候，直接用 `for in` 去遍历 viewControllers 即可:

```objc
    for (UIViewController viewController in self.navigationController.viewControllers){
        if ([viewController isKindOfClass:[AccountManageViewController class]]){
            [self.navigationController popToViewController:viewController animated:YES];
        }
    }
```



### 一个规范的 ViewController 的代码结构

一个 ViewController 中的代码如果不分类，那么查看起来就会非常混乱，经常需要到处跳转。因此写代码的时候按照顺序来分配代码块的位置。先是 `life cycle`，然后是`Delegate方法实现`，然后是`event response`，然后才是`getters and setters`，用 `#pragma mark - ` 分隔开。这样后来者阅读代码时就能省力很多。

> getter和setter全部都放在最后

因为一个ViewController很有可能会有非常多的view，如果getter和setter写在前面，就会把主要逻辑扯到后面去，其他人看的时候就要先划过一长串getter和setter，这样不太好。

> 每一个delegate都把对应的protocol名字带上，delegate方法不要到处乱写，写到一块区域里面去

比如UITableViewDelegate的方法集就老老实实写上`#pragma mark - UITableViewDelegate`。这样有个好处就是，当其他人阅读一个他并不熟悉的Delegate实现方法时，他只要按住command然后去点这个protocol名字，Xcode就能够立刻跳转到对应这个Delegate的protocol定义的那部分代码去，就省得他到处找了。

> event response专门开一个代码区域

所有button、gestureRecognizer的响应事件都放在这个区域里面，不要到处乱放。

> 关于private methods，正常情况下ViewController里面不应该写

不是delegate方法的，不是event response方法的，不是life cycle方法的，就是private method了。对的，正常情况下ViewController里面一般是不会存在private methods的，这个private methods一般是用于日期换算、图片裁剪啥的这种小功能。这种小功能要么把它写成一个category，要么把他做成一个模块，哪怕这个模块只有一个函数也行。

ViewController基本上是大部分业务的载体，本身代码已经相当复杂，所以跟业务关联不大的东西能不放在ViewController里面就不要放。另外一点，这个private method的功能这时候只是你用得到，但是将来说不定别的地方也会用到，一开始就独立出来，有利于将来的代码复用。

### 一些宏

#### #define

为一段代码设置一个缩写，这段代码可以是一个数字，也可以是一个函数。

```objc
#define SERVER_ONLINE_DEVELOPMENT
```

#### #if DEBUG

通过 `Build Configuration` 设置调试版本，分为 `debug` 和 `release`。正式发行的版本和 `release` 一致（但是真机调试的时候还是用develop证书）。可以使用预设宏来为开发环境添加一些额外的操作：

```objc
    #if DEBUG
    #define HttpDefaultURL HttpFormal
    #else
    #define HttpDefaultURL HttpSimulation
    #endif
```

这样就不用自己设置环境了。

如果要设置非 `debug`，那么使用 `#if !DEBUG`

#### #ifdef

该宏判断后面的宏是否被定义过：

```objc
#ifdef SERVER_ONLINE_DEVELOPMENT
	...
#endif
```

### 如何定义一个结构

使用结构保存多个相关数据。

```objc
struct Person{
	float height；
	int age；
}；
```

使用：`struct Person mikey；` 

使用 `typedef` 简化。`typedef` 可以为某类型声明一个等价的别名，该别名用法和常规数据类型无异。

```objc
typedef struct {
	float height；
	int age；
} Person；
```

使用，就当做一个普通类型使用：

```objc
Person mikey；
mikey.length = 1.45
```



### extern和static

我们知道在 `@interface...@end`,`@implementation...@end` 之外的是全局变量。OC 提供了两个关键字 `static` 和 `extern` 修饰。 

#### static

用 `static` 声明局部变量（定义在方法内），使其变为静态存储方式(静态数据区)，作用域不变（**其他方法不能使用**），但是延长了生命周期（程序结束才销毁）

用 `static` 声明外部变量（在方法外），使其**只在本文件内部有效**，而其他文件不可连接或引用该变量。

##### 变量

```objc
// 方法中或者 @implementation...@end 之外
static int a = 789;
```

`static` 可以在任意位置声明。一般我们都将其声明在 `.m` 文件中。可以声明在方法中，或者 `@implementation...@end` 之外。

##### 方法

```objc
static int method() {
    return 1;
}
```

`static` 的方法可以放在 `@implementation...@end` 之内，也可以放在之外。效果都是一样的。

#### extern

`static` 修饰的全局变量只能在该文件内访问，那么文件如何访问别的文件定义的全局变量呢？使用 `extern` 关键字。使用 `extern` 意味着**该变量或者方法在别处实现**。(你必须先用 `extern` 声明一下，这是为了告诉编译器进行全局变量的链接，否则直接编译的时候就会报错。)

如果全局变量和局部变量重名，则在局部变量作用域内，全局变量被屏蔽，不起作用。编程时候尽量不使用全局变量。

##### 变量

```objc
// 任意的某一个 .m 的 @implementation...@end 之外
NSString *s = @"global";

// 要用到全局变量的 .h 中
extern NSString *s;
```

在任意的一个 `.m` 中定义一个全局变量。然后要用到这个全局变量的地方使用 `extern`关键字即可。编译的时候就会链接到这个全局变量。

 `extern` 的变量写在任意位置都是可以的，包括 `.h` 的 `@interface...@end` 之内或之外或者`.m` 的 `@implementation...@end` 的之内之外。不过推荐写在 `.h` 的 `@interface...@end` 之外。

##### 方法

```objc
// 任意的 .m 中
int method() {
    return 1;
}

// 要用到的全局变量的 .h 中
extern method();
```

全局方法写在 `.m` 的 `@implementation...@end` 之内还是之外都行，因为这种方式定义的方法就已经表明它是全局方法了。

### 添加 UIGestureRecognizer 点击事件不响应

1. 设置 `UIView` 的 `userInteractionEnabled` 属性值为 `YES`，否则 `UIView` 会忽略那些原本应该发生在其自身的诸如 touch 和 keyboard 等用户事件，并将这些事件从消息队列中移除出去。
2. 循环创建对象的时候，不能只创建一个 `UIGestrueRecognizer`，而要在每个循环里为每个 `UIView` 创建并添加 `UITapGestureRecognizer` 。
3. 父view 太小，子view 通过 `addSubView` 的方式添加到父view 外部，导致响应链无法传递到子 view 中，无法响应 `UIGestureRecognizer`（被这个坑过很久）




### 倒计时按钮（每秒触发）

通过 gcd 设置一个每秒都会触发执行的队列，然后自己控制时间，何时退出。

```objc
// 开启倒计时效果
-(void)openCountdown{

    __block NSInteger time = 59; //倒计时时间

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

    dispatch_source_set_timer(_timer,dispatch_walltime(NULL, 0),1.0*NSEC_PER_SEC, 0); //每秒执行

    dispatch_source_set_event_handler(_timer, ^{

        if(time <= 0){ //倒计时结束，关闭

            dispatch_source_cancel(_timer);
            dispatch_async(dispatch_get_main_queue(), ^{

                //设置按钮的样式
                [self.authCodeBtn setTitle:@"重新发送" forState:UIControlStateNormal];
                [self.authCodeBtn setTitleColor:[UIColor colorFromHexCode:@"FB8557"] forState:UIControlStateNormal];
                self.authCodeBtn.userInteractionEnabled = YES;
            });

        }else{

            int seconds = time % 60;
            dispatch_async(dispatch_get_main_queue(), ^{

                //设置按钮显示读秒效果
                [self.authCodeBtn setTitle:[NSString stringWithFormat:@"重新发送(%.2d)", seconds] forState:UIControlStateNormal];
                [self.authCodeBtn setTitleColor:[UIColor colorFromHexCode:@"979797"] forState:UIControlStateNormal];
                self.authCodeBtn.userInteractionEnabled = NO;
            });
            time--;
        }
    });
    dispatch_resume(_timer);
}
```

注意：

> 我们在创建Button时, 要设置Button的样式:
> 当type为: UIButtonTypeCustom时 , 是读秒的效果.
> 当type为: 其他时, 是一闪一闪的效果.

### 获取float的前两位小数

当我们直接把 `float` 转为 `NSString`  会发现小数点后有很多小数，可以按照需求截取几位小数：

```objc
NSString *strDistance=[NSString stringWithFormat:@"%.xf", kilometers]; //x表示具体显示多少位
```
















​	