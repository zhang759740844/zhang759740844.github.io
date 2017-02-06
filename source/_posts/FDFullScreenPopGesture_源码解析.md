title: FDFullScreenPopGesture 源码解析
date: 2017/2/6 14:07:12  
categories: iOS
tags: 

	- 学习笔记
---

`FDFullScreenPopGesture` 是一个非常小巧精悍，却又功能强大的开源项目。主要的功能有：全局的手势回退；部分界面的 `NavigationBar` 的隐藏。今天花了半天的时间搞明白了其中的原理。

<!--more-->

iOS 原生的返回手势的起始滑动点必须靠近屏幕的左方，无法实现全屏返回。我们可以给控制器的 view 添加手势来控制它的 frame。但是那样需要自己控制动画，自己控制 `NavigationBar` ，非常麻烦。`FDFullScreenPopGesture` 通过一种优雅的方式解决了这个问题。并且顺带实现了部分控制器隐藏 `NavigationBar` 的功能。



### 总体解构

整个 `FDFullScreenPopGesture` 总共分为四个部分：

- `_FDFullscreenPopGestureRecognizerDelegate`
- `UIViewController (FDFullscreenPopGesturePrivate)`
- `UIViewController (FDFullscreenPopGesture)`
- `UINavigationController (FDFullscreenPopGesture)`

###  _FDFullscreenPopGestureRecognizerDelegate

这个类只有一个方法，就是实现了 `UIGestureRecognizerDelegate` 的代理方法 `gestureRecognizerShouldBegin:`。用来控制是否开始手势操作。

```objc
- (BOOL)gestureRecognizerShouldBegin:(UIPanGestureRecognizer *)gestureRecognizer
{
    // Ignore when no view controller is pushed into the navigation stack.
    if (self.navigationController.viewControllers.count <= 1) {
        return NO;
    }
    
    // Ignore when the active view controller doesn't allow interactive pop.
    UIViewController *topViewController = self.navigationController.viewControllers.lastObject;
    if (topViewController.fd_interactivePopDisabled) {
        return NO;
    }
    
    // Ignore when the beginning location is beyond max allowed initial distance to left edge.
    CGPoint beginningLocation = [gestureRecognizer locationInView:gestureRecognizer.view];
    CGFloat maxAllowedInitialDistance = topViewController.fd_interactivePopMaxAllowedInitialDistanceToLeftEdge;
    if (maxAllowedInitialDistance > 0 && beginningLocation.x > maxAllowedInitialDistance) {
        return NO;
    }

    // Ignore pan gesture when the navigation controller is currently in transition.
    if ([[self.navigationController valueForKey:@"_isTransitioning"] boolValue]) {
        return NO;
    }
   
    // Prevent calling the handler when the gesture begins in an opposite direction.
    CGPoint translation = [gestureRecognizer translationInView:gestureRecognizer.view];
    if (translation.x <= 0) {
        return NO;
    }
    
    return YES;
}
```

总共在5中情况下，不能全屏幕返回：

1. 整儿 `NavigationController` 中只有一个 `ViewController` 的时候
2. 当前的 `ViewController` 禁用了 `fd_interactivePopDisabled`（这个在 `UINavigationController (FDFullscreenPopGesture)` 中设置的）
3. 当前手势的启动点距离左边缘远过了 `fd_interactivePopMaxAllowedInitialDistanceToLeftEdge`（同上，也是自己设置的）
4. 当前是否在转场过程中。这里通过 KVC 拿到了 `NavigationController` 中的 `_isTransitioning` 属性
5. 手势运动方向是否是从右往左

### **UIViewController (FDFullscreenPopGesturePrivate)**

如果你对 `runtime` 比较熟悉，那么看到这个类一定非常亲切。这个方法在 `load` 方法的时候，通过 `Method Swizzling` 用 `fd_viewWillAppear:` 方法，替换了原来的 `viewWillAppear:` 方法。

```objc
- (void)fd_viewWillAppear:(BOOL)animated
{
    // Forward to primary implementation.
    [self fd_viewWillAppear:animated];
    
    if (self.fd_willAppearInjectBlock) {
        self.fd_willAppearInjectBlock(self, animated);
    }
}
```

这个替换的方法没什么特别的，就是在其中插入了自己的回调块 `fd_willAppearInjectBlock` ，该块在 `UINavigationController (FDFullscreenPopGesture)` 中定义并设置。该块用来设置 `NavigationBar` 的显示与隐藏。

###  UIViewController (FDFullscreenPopGesture)

这个类中定义了3个属性，上面用到了两个 `fd_interactivePopDisabled`，`fd_interactivePopMaxAllowedInitialDistanceToLeftEdge` 还有一个 `fd_prefersNavigationBarHidden` 属性用于判断是否隐藏 `NavigationBar` 上面的 `fd_willAppearInjectBlock` 中用到。

```objc
@interface UIViewController (FDFullscreenPopGesture)

/// Whether the interactive pop gesture is disabled when contained in a navigation
/// stack.
@property (nonatomic, assign) BOOL fd_interactivePopDisabled;

/// Indicate this view controller prefers its navigation bar hidden or not,
/// checked when view controller based navigation bar's appearance is enabled.
/// Default to NO, bars are more likely to show.
@property (nonatomic, assign) BOOL fd_prefersNavigationBarHidden;

/// Max allowed initial distance to left edge when you begin the interactive pop
/// gesture. 0 by default, which means it will ignore this limit.
@property (nonatomic, assign) CGFloat fd_interactivePopMaxAllowedInitialDistanceToLeftEdge;

@end
```



### UINavigationController (FDFullscreenPopGesture)

这个方法是整个项目的逻辑的核心，首先还是在 `load` 方法中用 `fd_pushViewController:animated:` 方法替换了系统自身的 `pushViewController:animated:` 方法。该方法如下：

```objc
- (void)fd_pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    if (![self.interactivePopGestureRecognizer.view.gestureRecognizers containsObject:self.fd_fullscreenPopGestureRecognizer]) {

        // 1.
        [self.interactivePopGestureRecognizer.view addGestureRecognizer:self.fd_fullscreenPopGestureRecognizer];

        // 2.
        NSArray *internalTargets = [self.interactivePopGestureRecognizer valueForKey:@"targets"];
        id internalTarget = [internalTargets.firstObject valueForKey:@"target"];
        SEL internalAction = NSSelectorFromString(@"handleNavigationTransition:");
        self.fd_fullscreenPopGestureRecognizer.delegate = self.fd_popGestureRecognizerDelegate;
        [self.fd_fullscreenPopGestureRecognizer addTarget:internalTarget action:internalAction];

        // 3.
        self.interactivePopGestureRecognizer.enabled = NO;
    }

    // 4.
    [self fd_setupViewControllerBasedNavigationBarAppearanceIfNeeded:viewController];

    // Forward to primary implementation.
    if (![self.viewControllers containsObject:viewController]) {
        [self fd_pushViewController:viewController animated:animated];
    }
}
```

先看第一步，在 `interactivePopGestureRecognizer` 的 view 上添加一个 `UIPanGestureRecognizer`。这个 `interactivePopGestureRecognizer` 是什么呢？ 这是控制界面在边缘处滑动返回的手势。这是一个只读的属性，我们当然不能直接设置它，拿到它是为了通过它拿到该手势所在的 view，在 view 中保存了一个 `_gestureRecognizers` 的数组。试想一下，如何判断是否需要 pop 呢？就是通过拿到 view 中的  `_gestureRecognizers` ，再调用每个手势的 `UIGestureRecognizerDelegate` 中的 gestureRecognizerShouldBegin` 方法是否返回 `yes`。这里苹果之所以把 `_gestureRecognizers` 设置为数组，可能是为了以后更易拓展吧。当然，这也方便了我们的拓展。

第二步将自己添加的手势设置成与原生手势相同，包括一个 `target` 和一个 `action`，这样两个手势的动画以及触发方法就一模一样了。

第三步将原来的手势禁用。

第四步设置上面说过的 `_FDViewControllerWillAppearInjectBlock` 。也就是通过 `fd_prefersNavigationBarHidden` 来显示和隐藏 `NavigationBar`。



### 总结

主要复习了一下 `method swizzling` 的使用，可以很优雅地修改特定的方法。



