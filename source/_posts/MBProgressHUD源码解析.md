title: MBProgressHUD 源码解析
date: 2017/2/21 14:07:12  
categories: iOS
tags: 

	- 源码解析


------

读项目工程的时候读到了 `MBProgressHUD`，所以仔细研究了一下它的原理，以及里面使用到的各种 api。我所看的源码是 **MBProgressHUD 1.0.0**

<!--more-->

## 构造与显示方法

首先是 `MBProgressHUD` 的构造显示方法：

```objc
+ (instancetype)showHUDAddedTo:(UIView *)view animated:(BOOL)animated {
    MBProgressHUD *hud = [[self alloc] initWithView:view];
    hud.removeFromSuperViewOnHide = YES;
    [view addSubview:hud];
    [hud showAnimated:animated];
    return hud;
}
```

该方法先创建了一个 `MBProgressHUD` 实例，然后将其添加到 view 上，并将其显示出来。`removeFromSuperViewOnHide` 是个标记，暂时不用管它。

### 构造方法

进入上面的 `initWithView:` 方法，一层层进入，最终通过 `commonInit` 实现：

```objc
- (void)commonInit {
    // Set default values for properties
    _animationType = MBProgressHUDAnimationFade;
    _mode = MBProgressHUDModeIndeterminate;
    _margin = 20.0f;
    _opacity = 1.f;
    _defaultMotionEffectsEnabled = YES;

    // Default color, depending on the current iOS version
    BOOL isLegacy = kCFCoreFoundationVersionNumber < kCFCoreFoundationVersionNumber_iOS_7_0;
    _contentColor = isLegacy ? [UIColor whiteColor] : [UIColor colorWithWhite:0.f alpha:0.7f];
    // Transparent background
    self.opaque = NO;
    self.backgroundColor = [UIColor clearColor];
    // Make it invisible for now
    self.alpha = 0.0f;
    self.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    //--- 父控件的透明度是否应用到子控件
    self.layer.allowsGroupOpacity = NO; 

    [self setupViews];
    [self updateIndicators];
    [self registerForNotifications];
}
```

这个方法构造了整个 view。上部设置了很多默认项，还是等到用到的时候再细说。后面又调用了三个方法 `setupViews`,`updateIndicators`,`registerForNotifications`。

#### 创建 view

进入 `setupVies` 方法：

```objc
- (void)setupViews {
    UIColor *defaultColor = self.contentColor;
    //---这个的背景view
    MBBackgroundView *backgroundView = [[MBBackgroundView alloc] initWithFrame:self.bounds];
    backgroundView.style = MBProgressHUDBackgroundStyleSolidColor;
    backgroundView.backgroundColor = [UIColor clearColor];
    backgroundView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    backgroundView.alpha = 0.f;
    [self addSubview:backgroundView];
    _backgroundView = backgroundView;
    //---容纳指示器的view
    MBBackgroundView *bezelView = [MBBackgroundView new];
    bezelView.translatesAutoresizingMaskIntoConstraints = NO;
    bezelView.layer.cornerRadius = 5.f;
    bezelView.alpha = 0.f;
    [self addSubview:bezelView];
    _bezelView = bezelView;
    //--- 这个方法让view能够跟随陀螺仪运动。
    [self updateBezelMotionEffects];

	...创建各种 label，detailLabel，button

    for (UIView *view in @[label, detailsLabel, button]) {
        view.translatesAutoresizingMaskIntoConstraints = NO;
        [view setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisHorizontal];
        [view setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisVertical];
        [bezelView addSubview:view];
    }

    UIView *topSpacer = [UIView new];
    topSpacer.translatesAutoresizingMaskIntoConstraints = NO;
    topSpacer.hidden = YES;
    [bezelView addSubview:topSpacer];
    _topSpacer = topSpacer;

    UIView *bottomSpacer = [UIView new];
    bottomSpacer.translatesAutoresizingMaskIntoConstraints = NO;
    bottomSpacer.hidden = YES;
    [bezelView addSubview:bottomSpacer];
    _bottomSpacer = bottomSpacer;
}
```

这个方法里创建了 HUD 中要显示的各个部分。首先创建了两个 `MBBackgroundView`，一个是整个 view 的背景，另一个则用来容纳指示器。进入它的构造方法，在 `updateForBackgroundStyle` 中用到了 `UIBlurEffect`。这是 iOS 中提供的用来实现高斯模糊的 API，不熟悉的朋友可以参见 [UIVisualEffectView 实现高斯模糊](https://zhang759740844.github.io/2017/02/13/iOS中API的使用方法/)

接下来是 `updateBezelMotionEffects` 方法。这个方法能够让 view 跟随屏幕倾斜而位移(看源码前还真没发现可以动)。这个方法里用到的是 `UIInterpolatingMotionEffect` 的 API，不熟悉的朋友可以参见 [UIInterpolatingMotionEffect 视图运动](https://zhang759740844.github.io/2017/02/13/iOS中API的使用方法/)

后面就是创建各个提示内容的 view，也就是后面经常用到的 `NSMutableArray *subviews = [NSMutableArray arrayWithObjects:self.topSpacer, self.label, self.detailsLabel, self.button, self.bottomSpacer, nil];`这里只是将 view 加入到 `MBBackgroundView` 中，约束设置在 `updateConstraints` 方法中，如果你对这个方法不太熟悉，可以参见[图像显示过程与一些注意事项](https://zhang759740844.github.io/2017/02/09/iOS技巧/)。

#### 设置指示器

现在执行到 `setupViews` 方法：

```objc
- (void)updateIndicators {
    UIView *indicator = self.indicator;
    BOOL isActivityIndicator = [indicator isKindOfClass:[UIActivityIndicatorView class]];
    BOOL isRoundIndicator = [indicator isKindOfClass:[MBRoundProgressView class]];

    MBProgressHUDMode mode = self.mode;
    if (mode == MBProgressHUDModeIndeterminate) {
        if (!isActivityIndicator) {
            //--- 系统自带的旋转的菊花。
            // Update to indeterminate indicator
            [indicator removeFromSuperview];
            indicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleWhiteLarge];
            [(UIActivityIndicatorView *)indicator startAnimating];
            [self.bezelView addSubview:indicator];
        }
    }
    else if (mode == MBProgressHUDModeDeterminateHorizontalBar) {
        // Update to bar determinate indicator
        //--- 自定义的一个进度条 条状的
        [indicator removeFromSuperview];
        indicator = [[MBBarProgressView alloc] init];
        [self.bezelView addSubview:indicator];
    }
    else if (mode == MBProgressHUDModeDeterminate || mode == MBProgressHUDModeAnnularDeterminate) {
        //--- 自定义进图条 条状的还是环状的
        if (!isRoundIndicator) {
            // Update to determinante indicator
            [indicator removeFromSuperview];
            indicator = [[MBRoundProgressView alloc] init];
            [self.bezelView addSubview:indicator];
        }
        if (mode == MBProgressHUDModeAnnularDeterminate) {
            [(MBRoundProgressView *)indicator setAnnular:YES];
        }
    } 
    else if (mode == MBProgressHUDModeCustomView && self.customView != indicator) {
        //--- 通过用户设置的 customView 设置进度条
        // Update custom view indicator
        [indicator removeFromSuperview];
        indicator = self.customView;
        [self.bezelView addSubview:indicator];
    }
    else if (mode == MBProgressHUDModeText) {
        [indicator removeFromSuperview];
        indicator = nil;
    }
    indicator.translatesAutoresizingMaskIntoConstraints = NO;
    self.indicator = indicator;

    if ([indicator respondsToSelector:@selector(setProgress:)]) {
        [(id)indicator setValue:@(self.progress) forKey:@"progress"];
    }

    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisHorizontal];
    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisVertical];

    [self updateViewsForColor:self.contentColor];
    [self setNeedsUpdateConstraints];
}
```

这个方法设置指示器到底是什么样的，通过前面设置的 mode 来区别。可以是系统自带的菊花型 `UIActivityIndicatorView`，也可以是自定义的条状指示器 `MBBarProgressView`，还可以是自定义的环状的 `MBRoundProgressView`，当然，也可以是用户自定义的 view 作为视图指示器。

如果是自定义的指示器，外部可以随时改变 customView，这会触发内部的 set 方法，更新约束：

```objc
- (void)setCustomView:(UIView *)customView {
    if (customView != _customView) {
        _customView = customView;
        if (self.mode == MBProgressHUDModeCustomView) {
            [self updateIndicators];
        }
    }
}
```

可以把 `customView` 设置为 `UIImageView` 来实现加载时的动态效果，详情看[用 UIImageView 播放动图](https://zhang759740844.github.io/2017/02/09/iOS技巧/)

在 `MBBarProgressView` 和 `MBRoundProgressView` 的 `drawRect:` 方法中实现了指示器的绘制。使用了 `UIBezierPath` 和 `CoreGraphics` 的 API，如果不熟，可以参见 [绘制图形](https://zhang759740844.github.io/2017/02/13/iOS中API的使用方法/)。进度显示是以 `_progress` 变量为准的。

#### 注册通知

这里的注册通知其实挺无关紧要的：

```objc
- (void)registerForNotifications {
#if !TARGET_OS_TV
    NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];

    [nc addObserver:self selector:@selector(statusBarOrientationDidChange:)
               name:UIApplicationDidChangeStatusBarOrientationNotification object:nil];
#endif
}
```

其实就是注册一个监听屏幕旋转的通知。而且现在大部分机型都是 iOS8 之后，基本不用设置什么。

### 显示方法

看一下显示方法 `showAnimated:`：

```objc
- (void)showAnimated:(BOOL)animated {
    MBMainThreadAssert();
    [self.minShowTimer invalidate];
    self.useAnimation = animated;
    self.finished = NO;
    // If the grace time is set, postpone the HUD display
    if (self.graceTime > 0.0) {
        NSTimer *timer = [NSTimer timerWithTimeInterval:self.graceTime target:self selector:@selector(handleGraceTimer:) userInfo:nil repeats:NO];
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        self.graceTimer = timer;
    } 
    // ... otherwise show the HUD immediately
    else {
        [self showUsingAnimation:self.useAnimation];
    }
}
```

可以看一下 `graceTime` 是一个显示的延迟时间。如果设置了这个时间，就会创建一个 `NSTimer` 的定时器，在 `graceTime` 之后再执行 `showUsingAnimation` 方法，将 hud 显示出来。`finished` 就是用来监控在这段时间还没有被显示出来的时间内，hud 是否已经被设置隐藏了的标记。

接下来看 `showUsingAnimation:` 方法：

```objc
- (void)showUsingAnimation:(BOOL)animated {
    // Cancel any previous animations
    [self.bezelView.layer removeAllAnimations];
    [self.backgroundView.layer removeAllAnimations];

    // Cancel any scheduled hideDelayed: calls
    [self.hideDelayTimer invalidate];
    //--- 记录显示的时间
    self.showStarted = [NSDate date];
    self.alpha = 1.f;

    // Needed in case we hide and re-show with the same NSProgress object attached.
    //--- 设置以CADisplayLink为基准的刷新，注意这里并不是所有的进度条都是以CAD来刷新的，只有设置了 progressObject 的才会使用
    [self setNSProgressDisplayLinkEnabled:YES];

    if (animated) {
        [self animateIn:YES withType:self.animationType completion:NULL];
    } else {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
        self.bezelView.alpha = self.opacity;
#pragma clang diagnostic pop
        self.backgroundView.alpha = 1.f;
    }
}
```

这里 `showStarted` 用来记录一个显示开始的时间，这个将会在后面用到，将通过这个时间来设置 hud 的最短显示时间。下面两个方法 `setNSProgressDisplayLinkEnabled:` 和 `animateIn:withType:completion:` 方法，一个用来设置 `progressObject` 进度（注意，不是所有进度变化都会用到这个方法），一个用来自定义展示动画。

我们来看看设置 progressObject 进度的方法：

```objc
- (void)setNSProgressDisplayLinkEnabled:(BOOL)enabled {
    // We're using CADisplayLink, because NSProgress can change very quickly and observing it may starve the main thread,
    // so we're refreshing the progress only every frame draw
    if (enabled && self.progressObject) {
        // Only create if not already active.
        if (!self.progressObjectDisplayLink) {
            //--- 这里要注意，重写了 progressObjectDisplayLink 的 set 方法
            self.progressObjectDisplayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(updateProgressFromProgressObject)];
        }
    } else {
        self.progressObjectDisplayLink = nil;
    }
}

- (void)updateProgressFromProgressObject {
    self.progress = self.progressObject.fractionCompleted;
}
```

之所以说是设置 `progressObject` 的进度，是因为显示的进度是以 `_progress` 为准的。如果你仔细看上面的 `drawRect:` 方法，就能发现所有的图形都是以 `_progress` 显示的。那么这里的 `progressObject` 有什么用呢？

如果使用了 `progressObject`，那么就会创建一个 `CADisplayLink` 实例。`CADisplayLink` 适合于高精度的定时刷新，具体使用方法可见：[CADisplayLink 方式的定时器](https://zhang759740844.github.io/2017/02/13/iOS中API的使用方法/) 

`progressObject` 是一个 `NSProgress` 对象，如果对其不了解可以参见[iOS进度指示器——NSProgress](https://my.oschina.net/u/2340880/blog/679367)，它将进度设置给 `_progress`。我们可以看一下 `_progress` 的 set 方法：

```objc
- (void)setProgress:(float)progress {
    if (progress != _progress) {
        _progress = progress;
        UIView *indicator = self.indicator;
        if ([indicator respondsToSelector:@selector(setProgress:)]) {
            [(id)indicator setValue:@(self.progress) forKey:@"progress"];
        }
    }
}
```

这样就会触发指示器的 `progress` 的 set 方法：

```objc
- (void)setProgress:(float)progress {
    if (progress != _progress) {
        _progress = progress;
        [self setNeedsDisplay];
    }
}
```

只要 `progress` 变化了，就会设置重绘视图。

至于动画的执行方法，没什么特别的地方，不太了解的话可以参考 [核心动画](https://zhang759740844.github.io/2016/08/16/ios动画/)

## 隐藏方法

来看一下隐藏方法：

```objc
+ (BOOL)hideHUDForView:(UIView *)view animated:(BOOL)animated {
    MBProgressHUD *hud = [self HUDForView:view];
    if (hud != nil) {
        hud.removeFromSuperViewOnHide = YES;
        [hud hideAnimated:animated];
        return YES;
    }
    return NO;
}
```

先找到 `MBProgressHUD` 类，然后调动 `hideAnimated:` 方法：

```objc
- (void)hideAnimated:(BOOL)animated {
    MBMainThreadAssert();
    [self.graceTimer invalidate];
    self.useAnimation = animated;
    self.finished = YES;
    // If the minShow time is set, calculate how long the HUD was shown,
    // and postpone the hiding operation if necessary
    if (self.minShowTime > 0.0 && self.showStarted) {
        NSTimeInterval interv = [[NSDate date] timeIntervalSinceDate:self.showStarted];
        if (interv < self.minShowTime) {
            NSTimer *timer = [NSTimer timerWithTimeInterval:(self.minShowTime - interv) target:self selector:@selector(handleMinShowTimer:) userInfo:nil repeats:NO];
            [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
            self.minShowTimer = timer;
            return;
        } 
    }
    // ... otherwise hide the HUD immediately
    [self hideUsingAnimation:self.useAnimation];
}
```

这个方法的解构和上面的 show 类似，其中 `minShowTime` 表示最少显示的时间，避免加载太快，hud 一闪而过的情况出现。这里就用到了上面说过的 `showStarted` 了。往下走，来看看 `hideUsingAnimation:` 方法：

```objc
- (void)hideUsingAnimation:(BOOL)animated {
    if (animated && self.showStarted) {
        self.showStarted = nil;
        [self animateIn:NO withType:self.animationType completion:^(BOOL finished) {
            [self done];
        }];
    } else {
        self.showStarted = nil;
        self.bezelView.alpha = 0.f;
        self.backgroundView.alpha = 1.f;
        [self done];
    }
}
```

其实也和 show 的时候差不多，有动画显示隐藏动画，没有动画直接就隐藏掉。只不过这里要做一些扫尾操作：执行 `done` 方法：

```objc
- (void)done {
    // Cancel any scheduled hideDelayed: calls
    [self.hideDelayTimer invalidate];
    [self setNSProgressDisplayLinkEnabled:NO];

    if (self.hasFinished) {
        self.alpha = 0.0f;
        if (self.removeFromSuperViewOnHide) {
            [self removeFromSuperview];
        }
    }
    MBProgressHUDCompletionBlock completionBlock = self.completionBlock;
    if (completionBlock) {
        completionBlock();
    }
    id<MBProgressHUDDelegate> delegate = self.delegate;
    if ([delegate respondsToSelector:@selector(hudWasHidden:)]) {
        [delegate performSelector:@selector(hudWasHidden:) withObject:self];
    }
}
```

这里就是执行一下外部传进来的 `completionBlock` 回调。如果外部设置了代理方法，并且重写了 `hudWasHidden` 那么就调用执行。



## 总结

`MBProgressHUD` 确实是一个比较简单的框架。看的时候我还是蛮仔细的注意每一个知识点的，诸如 view 的绘制显示过程、贝塞尔曲线的绘制、`UIInterpolatingMotionEffect` 的使用 等等等等。所以还是花了不少时间去搞懂的。写到这里再想想，整个框架的结构还是非常清晰易懂的。





 

