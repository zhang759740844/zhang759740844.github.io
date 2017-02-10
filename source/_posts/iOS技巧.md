title: 一些 iOS 小技巧与注意点(持续更新)
date: 2017/2/9 10:07:12  
categories: iOS
tags:
	- 学习笔记


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

在 `BaseViewController` 中添加一个 `reloadDataWithBlank` 方法：
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
其中 `blankHintView` 就是空白提示视图。每次在 `viewDidLoad` 中，或者在这个方法内再创建一个功能更明确的 `loadDefaultDataSource` 方法，在其中中添加默认的 `title` 和 `image` 即可。

### 设置手势事件
关于手势，通常需要写一个事件的方法。这样设置事件和方法本身就会分隔开来。回顾代码的时候找起来会很麻烦。可以通过 block 回调的方式将方法保存起来：
```objc
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

