title: ios核心动画
date: 2016/8/16 14:07:12  
categories: iOS
tags: [Animation]

-----

## 视图动画

###UIView动画

#### 首尾式动画

执行动画所需要的工作由UIView类自动完成，但仍要在希望执行动画时通知视图，为此需要将改变属性的代码放在 `[UIView beginAnimations:nil context:nil]` 和 `[UIView commitAnimations]` 之间。

常见方法：

```objc
// 动画开始时调用
+ (void)setAnimationWillStartSelector:(SEL)selector
// 动画结束时调用
+ (void)setAnimationDidStopSelector:(SEL)selector
// 动画的持续时间
+ (void)setAnimationDuration:(NSTimeInterval)duration
// 动画延迟delay秒后再开始
+ (void)setAnimationDelay:(NSTimeInterval)delay
// 动画加速减速效果
+ (void)setAnimationCurve:(UIViewAnimationCurve)curve
// 动画自动反向回到原来位置
+ (void)setAnimationRepeatAutoreverses:(BOOL)repeatAutoreverses
// 动画重复次数
+ (void)setAnimationRepeatCount:(float)repeatCount
```

#### block 动画

```objc
// 普通的动画
+ (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion;
// 带阻尼的动画
+ (void)animateWithDuration:(NSTimeInterval)duration delay:(NSTimeInterval)delay usingSpringWithDamping:(CGFloat)dampingRatio initialSpringVelocity:(CGFloat)velocity options:(UIViewAnimationOptions)options animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion;
```

#### 示例

```objc
//首尾式动画
[UIView beginAnimations:nil context:nil];
//设置动画执行时间
[UIView setAnimationDuration:2.0];
//设置动画执行完毕调用的事件
[UIView setAnimationDidStopSelector:@selector(didStopAnimation)];
self.customView.center=CGPointMake(200, 300);
[UIView commitAnimations];
```

### UIView 关键帧动画

关键帧动画使用 block 执行。包含了几个必要属性：

```objc
[UIView animateKeyframesWithDuration:1 delay:0.5 options:UIViewKeyframeAnimationOptionCalculationModeLinear animations:^{
    self.imageView.frame = CGRectMake(0, 0, 100, 100);
} completion:^(BOOL finished) {
    NSLog(@"结束了");
}];
```

既然是关键帧动画，那么肯定是能增加帧的。在原本执行动画的方法中通过增加帧：

```objc
[UIView animateKeyframesWithDuration:1 delay:0.5 options:UIViewKeyframeAnimationOptionCalculationModeLinear animations:^{
  
    [UIView addKeyframeWithRelativeStartTime:0.3 relativeDuration:0.3 animations:^{
        self.imageView.frame = CGRectMake(0, 0, 100, 100);
    }];
  
    [UIView addKeyframeWithRelativeStartTime:0.3 relativeDuration:0.3 animations:^{
        self.imageView.frame = CGRectMake(100, 100, 200, 200);
    }];
    
} completion:^(BOOL finished) {
    NSLog(@"结束了");
}];
```

### CALayer 关键帧动画

CALayer 的动画其实可以用 UIView 的动画来代替。不过 CALayer 的关键帧动画可以让视图沿着指定路径移动

#### 参数形式

```objc
//根据values移动的动画
CAKeyframeAnimation *catKeyAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
CGPoint originalPoint = self.catImageView.layer.frame.origin;
CGFloat distance =  50;

NSValue *value1 = [NSValue valueWithCGPoint:CGPointMake(originalPoint.x + distance, originalPoint.y + distance)];
NSValue *value2 = [NSValue valueWithCGPoint:CGPointMake(originalPoint.x + 2 * distance, originalPoint.y + distance)];
NSValue *value3 = [NSValue valueWithCGPoint:CGPointMake(originalPoint.x + 2 * distance, originalPoint.y +  2 * distance)];
NSValue *value4 = [NSValue valueWithCGPoint:originalPoint];
catKeyAnimation.values = @[value4, value1, value2, value3, value4];

catKeyAnimation.duration = 2;
catKeyAnimation.repeatCount = MAXFLOAT;
catKeyAnimation.removedOnCompletion = NO;
[self.catImageView.layer addAnimation:catKeyAnimation forKey:nil];
```

#### 路径形式

```objc
-(void)pathAnimation2 {
    //创建path
    UIBezierPath *path = [UIBezierPath bezierPath];
    //设置线宽
    path.lineWidth = 3;
    //线条拐角
    path.lineCapStyle = kCGLineCapRound;
    //终点处理
    path.lineJoinStyle = kCGLineJoinRound;
    //多条直线
    [path moveToPoint:(CGPoint){0, SCREEN_HEIGHT/2-50}];
    [path addLineToPoint:(CGPoint){SCREEN_WIDTH/3, SCREEN_HEIGHT/2-50}];
    [path addLineToPoint:(CGPoint){SCREEN_WIDTH/3, SCREEN_HEIGHT/2+50}];
    [path addLineToPoint:(CGPoint){SCREEN_WIDTH*2/3, SCREEN_HEIGHT/2+50}];
    [path addLineToPoint:(CGPoint){SCREEN_WIDTH*2/3, SCREEN_HEIGHT/2-50}];
    [path addLineToPoint:(CGPoint){SCREEN_WIDTH, SCREEN_HEIGHT/2-50}];
//    [path closePath];

    CAKeyframeAnimation *anima = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    anima.path = path.CGPath;
    anima.duration = 2.0f;
    anima.fillMode = kCAFillModeForwards;
    anima.removedOnCompletion = NO;
    anima.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    [_demoView.layer addAnimation:anima forKey:@"pathAnimation"];
}
```

### 转场动画

> 转场动画就是就是把下一个显示的时刻的界面截了个图，然后做转场。

示例：

```objc
// 先设置下一个场景
self.iconView.image=[UIImage imageNamed: [NSString stringWithFormat:@"%d.jpg",self.index]];

//创建核心动画
CATransition *ca=[CATransition animation];
//告诉要执行什么动画
//设置过度效果
ca.type=@"cube";
//设置动画的过度方向（向左）
ca.subtype=kCATransitionFromLeft;
//设置动画的时间
ca.duration=2.0;
//添加动画
[self.iconView.layer addAnimation:ca forKey:nil];
```

## 控制器动画

### Modal 动画

