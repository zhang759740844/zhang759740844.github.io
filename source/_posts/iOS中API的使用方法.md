title: 许多 iOS API 的使用方式(持续更新)
date: 2017/2/13 14:07:12  
categories: iOS
tags: 

	- 学习笔记
	- 持续更新

------

很多不常用的简单的 API 的使用方式。分开写的话太短小，就合在一起吧。（这里都是学到的时候，东找一点西找一点拼凑起来的，一开始没有记录出处在哪，如果哪里有侵权的地方，还请尽快告知呀。我会第一时间注明出处的。谢谢啦~）

<!--more-->

## UIView 中的坐标转换

一个 View 的 `frame` 的起点是相当于其所在的 View，即调用 `addSubView:` 方法的 View。如果要判断两个 View 是否是包含关系，由于两者的起点不同，那么肯定是无法进行比较的。

```objc
// rect1和rect2是否有重叠
CGRectContainsRect(<#CGRect rect1#>, <#CGRect rect2#>)
// point是不是在rect上
CGRectContainsPoint(<#CGRect rect#>, <#CGPoint point#>)
// rect1是否包含了rect2
CGRectIntersectsRect(<#CGRect rect1#>, <#CGRect rect2#>)
```

为了统一原点，我们可以使用以下代码：

```objc
- (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view;
- (CGPoint)convertPoint:(CGPoint)point fromView:(nullable UIView *)view;

- (CGRect)convertRect:(CGRect)rect toView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect fromView:(nullable UIView *)view;
```

来举两个例子，注意不同情况下 `compareView` 和 `outerView` 的参数位置：

```objc
CGRect newRect = [self.compareView convertRect:self.innerFrame fromView:self.outerView];
CGRect newRect = [self.outerView convertRect:self.innerFrame toView:self.compareView];
```

得到的就是 `innerFrame` 在 `compareView` 中的位置。







## UIVisualEffectView 实现高斯模糊

如果想要给一个 view  添加一个高斯模糊的效果，只要在那个 view 上添加一个 `UIVisualEffectView` 即可。

高斯模糊有三种效果，从浅入深的 style 依次是：

- UIBlurEffectStyleExtraLight
- UIBlurEffectStyleLight
- UIBlurEffectStyleDark



使用的基本实例：

```objc
// 要添加模糊的view
UIImageView *imageview = [[UIImageView alloc] init];
imageview.frame = CGRectMake(10, 100, 300, 300);
imageview.image = [UIImage imageNamed:@"2"];
imageview.contentMode = UIViewContentModeScaleAspectFit;
imageview.userInteractionEnabled = YES;
[self.view addSubview:imageview];

// 高斯模糊的view
UIBlurEffect *blur = [UIBlurEffect effectWithStyle:UIBlurEffectStyleLight];
UIVisualEffectView *effectview = [[UIVisualEffectView alloc] initWithEffect:blur];
effectview.frame = CGRectMake(0, 0, imageview.size.width/2, 300);

// 添加高斯模糊
[imageview addSubview:effectview];
```



## UIInterpolatingMotionEffect 视图运动

`UIInterpolatingMotionEffect` 可以通过陀螺仪监测手机的倾斜情况。我们可以通过它设置视图响应的运动。

其中有两个属性：`minimumRelativeValue` 和 `maximumRelativeValue`。这两个属性控制图像运动的最大范围。

实例：

```objc
CGFloat effectOffset = 100.f;
// 设置x方向上的移动量
UIInterpolatingMotionEffect *effectX = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.x" type:UIInterpolatingMotionEffectTypeTiltAlongHorizontalAxis];
effectX.maximumRelativeValue = @(effectOffset);
effectX.minimumRelativeValue = @(-effectOffset);

// 设置y方向上的移动量
UIInterpolatingMotionEffect *effectY = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.y" type:UIInterpolatingMotionEffectTypeTiltAlongVerticalAxis];
effectY.maximumRelativeValue = @(effectOffset);
effectY.minimumRelativeValue = @(-effectOffset);

// 将移动量添加到数组中
UIMotionEffectGroup *group = [[UIMotionEffectGroup alloc] init];
group.motionEffects = @[effectX, effectY];

// 设置给 view
[self.view addMotionEffect:group];
```



> 

