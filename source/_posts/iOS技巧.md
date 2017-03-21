title: 一些 iOS 小技巧与注意点(持续更新)
date: 2017/2/9 10:07:12  
categories: iOS
tags:
	- 学习笔记
	- 持续更新

------

这篇中将收集一些关于 iOS 开发中的一些小技巧以及注意事项，可能比较零散。

<!--more-->

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

自动布局将视图显示到屏幕上的步骤总共分为三步：

1. 更新约束。它为布局准备好必要的信息，而这些布局将在实际设置视图的 frame 时被传递过去并被使用。你可以通过调用  `setNeedsUpdateConstraints` 来触发这个操作。谈到自定义视图，**可以重写 `updateConstraints` 来为你的视图的本地约束进行修改**，但要确保在你的实现中修改了任何你需要布局子视图的约束条件**之后**，调用一下 `[super updateConstraints]`。
2. 布局。将约束条件应用到视图上。可以**通过重写 `layoutSubViews` 实现**，也就是在调用 `[super layoutSubviews]` 前修改一些约束信息。可以通过调用 `setNeedsLayout` 来触发一个操作请求，这并不会立刻应用布局，而是在稍后再进行处理。因为所有的布局请求将会被合并到一个布局操作中去。可以调用  `layoutIfNeeded` 来强制系统立即更新视图树的布局。
3. 显示。可以通过调用 `setNeedsDisplay` 来触发，这将会导致所有的调用都被合并到一起推迟重绘。重写熟悉的 `drawRect:` ，通过这个方法我们可以在 `UIView` 中绘制图形，比如绘制个角标之类的。

![图像显示流程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/display流程.png?raw=true)

#### `layoutSubViews` 的使用时机

任何地方都能够设置约束条件，那么我们为什么还要重写这个方法呢？

> 在苹果的官方文档中强调: You should override this method only if the autoresizing behaviors of the subviews do not offer the behavior you want.layoutSubviews

当我们在某个类的内部调整子视图位置时，需要调用。反过来的意思就是说：如果你想要在外部设置subviews的位置，就不要重写。

#### `updateConstraints` 的使用时机

任何地方都能够设置约束条件，只有在调用了 `layoutsubviews` 方法之后才会更新视图（调用了 `updateConstraints` 不会更新视图）。所以更新视图的动画方法一般设置为：

```objc
[UIView animateWithDuration:1.0f delay:0.0f usingSpringWithDamping:0.5f initialSpringVelocity:1 options:UIViewAnimationOptionCurveEaseInOut animations:^{
        [self.view layoutIfNeeded];
} completion:NULL];
```

**`updateConstraints` 在什么时候使用呢？**关于 `setNeedsUpdateConstraints` 文档上是这么说的：

> When a property of your custom view changes in a way that would impact constraints, you can call this method to indicate that the constraints need to be updated at some point in the future. The system will then call updateConstraints as part of its normal layout pass. Updating constraints all at once just before they are needed ensures that you don’t needlessly recalculate constraints when multiple changes are made to your view in between layout passes.

不要将 `updateConstraints` 用于视图的初始化设置。**当你需要在单个布局流程（single layout pass）中添加、修改或删除大量约束的时候，用它来获得最佳性能。如果没有性能问题，直接更新约束更简单。如果你不會在程式運行途中去新增或移除 constraints，那就把建立 constraints 的工作放在 `init` 或 `viewDidLoad` 階段。中途會新增或移除的動作才放到 `updateConstraints`。你可以建立 constraint 之後再修改它的 constant 屬性，這不算是新增或移除 constraint。** 移除约束的方法是：` [self removeConstraints:self.constraints];`。

#### 总结

对于手撸一个 view 来说，一般不用重写 `updateConstraints`，只有在某些情况要增删约束的时候才用到，约束信息放在 `init` 方法里添加就可以了。

view 中子视图的约束可以放在 `init` 中，但是设置子视图的 `bounds` 不能还放在该方法中。因为当外部创建该 view 时，如果外部调用 `init` 方法而不是调用 `initWithFrame` (即没有给这个 view 设置大小），那么 view 中子视图就无法正确初始化出大小。因此，子视图的 `bounds` 等属性（例如 `center` 等）的设置必须放在 `layoutSubviews` 中，即外部 view 大小已经确定，就要显示的时候。当然显示出来后，子视图的 `bounds` 再变化就和 `layoutSubviews` 方法没有任何关系了，在代码的任意位置修改都可以。

`drawRect` 方法通过 `CGContextRef context = UIGraphicsGetCurrentContext();` 拿到上下文句柄，然后将自己要画的图形添加到这个上下文上。

### 设置 view 的一些注意事项

- 当在代码中设置视图和它们的约束条件时候，一定要记得将该视图的 `translatesAutoResizingMaskIntoConstraints` 属性设置为 NO。如果忘记设置这个属性几乎肯定会导致不可满足的约束条件错误。
- 在 `initWithFrame:` 方法中将子控件加到 view 而不是设置尺寸。因为 view 有可能是通过 `init` 方法创建的，这个时候 view 的 frame 可能是 不确定的。这种情况下各个子控件的尺寸都会是0，因为这个 view 的 frame 还没有设置。设置尺寸的工作放在 `layoutSubviews` 中去做。
- `allowsGroupOpacity` 属性允许子控件的不透明度继承于其父控件，默认是开启的 `yes`。不过这会影响性能，自定义控件的时候最好设置为 `self.layer.allowsGroupOpacity = NO;`
- `clipsToBounds` 是 `UIView` 的属性，如果设置为 `yes`，则不显示超出父 View 的部分；`masksToBounds` 是 `CALayer` 的属性，如果设置为 `yes`，则不显示超出父 View layer 的部分.


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