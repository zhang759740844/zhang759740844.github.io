title: ios核心动画
date: 2016/8/16 14:07:12  
categories: IOS
tags: [Animation]

---
依旧是[文顶顶的ios开发UI篇](http://www.cnblogs.com/wendingding/tag/UI高级/)关于核心动画的内容。作为入门

**核心动画主要就是按照设定的规则(重复，延迟，持续时间等)，操作layer或者view的一些属性(position,bound,transform等)。**

<!--more-->

## 核心动画简介
CAAnimation是所有动画类的父类，但是它不能直接使用，能用的动画类只有4个子类：CABasicAnimation、CAKeyframeAnimation、CATransition、CAAnimationGroup。

CAPropertyAnimation是CAAnimation的子类，但是不能直接使用，要想创建动画对象，应该使用它的两个子类：CABasicAnimation和CAKeyframeAnimation
它有个NSString类型的keyPath属性，你可以指定CALayer的某个属性名为keyPath，并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@"position"为keyPath，就会修改CALayer的position属性的值，以达到平移的动画效果

常见属性：
- duration：动画的持续时间
- repeatCount：动画的重复次数
- repeatDuration：动画的重复时间
- removedOnCompletion：默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode为kCAFillModeForwards
- fillMode：决定当前对象在非active时间段的行为.比如动画开始之前,动画结束之后
- beginTime：可以用来设置动画延迟执行时间，若想延迟2s，就设置为CACurrentMediaTime()+2，CACurrentMediaTime()为图层的当前时间
- timingFunction：速度控制函数，控制动画运行的节奏
- delegate：动画代理

## 基础动画
### 简介
CABasicAnimation，是CApropertyAnimation的子类

属性：
- fromValue：keyPath相应属性的初始值
- toValue：keyPath相应属性的结束值

随着动画的进行，在长度为duration的持续时间内，keyPath相应属性的值从fromValue渐渐地变为toValue

如果fillMode=kCAFillModeForwards和removedOnComletion=NO，那么在动画执行完毕后，图层会保持显示动画执行后的状态。但**在实质上，图层的属性值还是动画执行前的初始值，并没有真正被改变。**比如，CALayer的position初始值为(0,0)，CABasicAnimation的fromValue为(10,10)，toValue为(100,100)，虽然动画执行完毕后图层保持在(100,100)这个位置，实质上图层的position还是为(0,0)。

### 示例
```objc
#import "YYViewController.h"

@interface YYViewController ()
@property(nonatomic,strong)CALayer *myLayer;
@end

@implementation YYViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    //创建layer
    CALayer *myLayer=[CALayer layer];
    //设置layer的属性
    myLayer.bounds=CGRectMake(0, 0, 50, 80);
    myLayer.backgroundColor=[UIColor yellowColor].CGColor;
    myLayer.position=CGPointMake(50, 50);
    myLayer.anchorPoint=CGPointMake(0, 0);
    myLayer.cornerRadius=20;
    //添加layer
    [self.view.layer addSublayer:myLayer];
    self.myLayer=myLayer;
}
//设置 平移 动画
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //1.创建核心动画
    //    CABasicAnimation *anima=[CABasicAnimation animationWithKeyPath:<#(NSString *)#>]
    CABasicAnimation *anima=[CABasicAnimation animation];
    
    //1.1告诉系统要执行什么样的动画
    anima.keyPath=@"position";
    //设置通过动画，将layer从哪儿移动到哪儿
    anima.fromValue=[NSValue valueWithCGPoint:CGPointMake(0, 0)];
    anima.toValue=[NSValue valueWithCGPoint:CGPointMake(200, 300)];
    
    //1.2设置动画执行完毕之后不删除动画
    anima.removedOnCompletion=NO;
    //1.3设置保存动画的最新状态
    anima.fillMode=kCAFillModeForwards;

    //2.添加核心动画到layer
    [self.myLayer addAnimation:anima forKey:nil];

}

-(void)animationDidStart:(CAAnimation *)anim
{
    NSLog(@"开始执行动画");
}

-(void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag
{
    //动画执行完毕，打印执行完毕后的position值
    NSString *str=NSStringFromCGPoint(self.myLayer.position);
    NSLog(@"执行后：%@",str);
}

@end
```

其中keypath的值决定产生什么动画
1. position：执行平移动画
2. bounds：执行缩放动画
3. transform：执行旋转动画

## 关键帧动画
### 简介
CAKeyframeAnimation，是CApropertyAnimation的子类。CABasicAnimation只能从一个数值(fromValue)变到另一个数值(toValue)，而CAKeyframeAnimation会使用一个NSArray保存这些数值

属性：
- values：就是上述的NSArray对象。里面的元素称为”关键帧”(keyframe)。动画对象会在指定的时间(duration)内，依次显示values数组中的每一个关键帧
- path：可以设置一个CGPathRef\CGMutablePathRef,让层跟着路径移动。path只对CALayer的anchorPoint和position起作用。如果你设置了path，那么values将被忽略
- keyTimes：可以为对应的关键帧指定对应的时间点,其取值范围为0到1.0,keyTimes中的每一个时间值都对应values中的每一帧.当keyTimes没有设置的时候,各个关键帧的时间是平分的

说明：CABasicAnimation可看做是最多只有2个关键帧的CAKeyframeAnimation

### 示例
#### 使用value
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //1.创建核心动画
    CAKeyframeAnimation *keyAnima=[CAKeyframeAnimation animation];
    //平移
    keyAnima.keyPath=@"position";
    //1.1告诉系统要执行什么动画
    NSValue *value1=[NSValue valueWithCGPoint:CGPointMake(100, 100)];
    NSValue *value2=[NSValue valueWithCGPoint:CGPointMake(200, 100)];
    NSValue *value3=[NSValue valueWithCGPoint:CGPointMake(200, 200)];
    NSValue *value4=[NSValue valueWithCGPoint:CGPointMake(100, 200)];
    NSValue *value5=[NSValue valueWithCGPoint:CGPointMake(100, 100)];
    keyAnima.values=@[value1,value2,value3,value4,value5];
    //1.2设置动画执行完毕后，不删除动画
    keyAnima.removedOnCompletion=NO;
    //1.3设置保存动画的最新状态
    keyAnima.fillMode=kCAFillModeForwards;
    //1.4设置动画执行的时间
    keyAnima.duration=4.0;
    //1.5设置动画的节奏
    keyAnima.timingFunction=[CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    //设置代理，开始—结束
    keyAnima.delegate=self;
    //2.添加核心动画
    [self.customView.layer addAnimation:keyAnima forKey:nil];
}
```

#### 使用path
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //1.创建核心动画
    CAKeyframeAnimation *keyAnima=[CAKeyframeAnimation animation];
    //平移
    keyAnima.keyPath=@"position";
    //1.1告诉系统要执行什么动画
    //创建一条路径
    CGMutablePathRef path=CGPathCreateMutable();
    //设置一个圆的路径
    CGPathAddEllipseInRect(path, NULL, CGRectMake(150, 100, 100, 100));
    keyAnima.path=path;
    
    //有create就一定要有release
    CGPathRelease(path);
    //1.2设置动画执行完毕后，不删除动画
    keyAnima.removedOnCompletion=NO;
    //1.3设置保存动画的最新状态
    keyAnima.fillMode=kCAFillModeForwards;
    //1.4设置动画执行的时间
    keyAnima.duration=5.0;
    //1.5设置动画的节奏
    keyAnima.timingFunction=[CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    //设置代理，开始—结束
    keyAnima.delegate=self;
    //2.添加核心动画
    [self.customView.layer addAnimation:keyAnima forKey:@"wendingding"];
}

- (IBAction)stopOnClick:(UIButton *)sender {
    //停止self.customView.layer上名称标示为wendingding的动画
    [self.customView.layer removeAnimationForKey:@"wendingding"];
}
```

点击停止动画，程序内部会调用  [self.customView.layer removeAnimationForKey:@"wendingding"];停止self.customView.layer上名称标示为wendingding的动画。

#### 图标抖动
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //1.创建核心动画
    CAKeyframeAnimation *keyAnima=[CAKeyframeAnimation animation];
    keyAnima.keyPath=@"transform.rotation";
    //设置动画时间
    keyAnima.duration=0.1;
    //设置图标抖动弧度
    //把度数转换为弧度  度数/180*M_PI
    keyAnima.values=@[@(-angle2Radian(4)),@(angle2Radian(4)),@(-angle2Radian(4))];
    //设置动画的重复次数(设置为最大值)
    keyAnima.repeatCount=MAXFLOAT;
    
    keyAnima.fillMode=kCAFillModeForwards;
    keyAnima.removedOnCompletion=NO;
    //2.添加动画
    [self.iconView.layer addAnimation:keyAnima forKey:nil];
}
```

其中，`keyAnima.values=@[@(-angle2Radian(4)),@(angle2Radian(4)),@(-angle2Radian(4))];`表示从-angle2Radian(4)转到angle2Radian(4)再转回-angle2Radian(4))。

**@()表示初始化oc对象，即将double转换为oc对象。因为oc的数组中不允许出现非oc得对象**

## 转场动画和组动画
### 介绍
CATransition用于做转场动画
属性：
- type：动画过渡类型
- subtype：动画过渡方向
- startProgress：动画起点(在整体动画的百分比)
- endProgress：动画终点(在整体动画的百分比)

### 示例
```objc
- (IBAction)preOnClick:(UIButton *)sender {
    self.index--;
    if (self.index<1) {
        self.index=7;
    }
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
}

//下一张
- (IBAction)nextOnClick:(UIButton *)sender {
    self.index++;
    if (self.index>7) {
        self.index=1;
    }
        self.iconView.image=[UIImage imageNamed: [NSString stringWithFormat:@"%d.jpg",self.index]];
    
    //1.创建核心动画
    CATransition *ca=[CATransition animation];
    
    //1.1告诉要执行什么动画
    //1.2设置过度效果
    ca.type=@"cube";
    //1.3设置动画的过度方向（向右）
    ca.subtype=kCATransitionFromRight;
    //1.4设置动画的时间
    ca.duration=2.0;
    //1.5设置动画的起点
    ca.startProgress=0.5;
    //1.6设置动画的终点
//    ca.endProgress=0.5;
    
    //2.添加动画
    [self.iconView.layer addAnimation:ca forKey:nil];
}
```

### 组动画
将CAAnimationGroup对象加入层后，组中所有动画对象可以同时并发运行属性解析.

### 示例
```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{
    // 平移动画
    CABasicAnimation *a1 = [CABasicAnimation animation];
    a1.keyPath = @"transform.translation.y";
    a1.toValue = @(100);
    // 缩放动画
    CABasicAnimation *a2 = [CABasicAnimation animation];
    a2.keyPath = @"transform.scale";
    a2.toValue = @(0.0);
    // 旋转动画
    CABasicAnimation *a3 = [CABasicAnimation animation];
    a3.keyPath = @"transform.rotation";
    a3.toValue = @(M_PI_2);
    // 组动画
    CAAnimationGroup *groupAnima = [CAAnimationGroup animation];
    
    groupAnima.animations = @[a1, a2, a3];
    
    //设置组动画的时间
    groupAnima.duration = 2;
    groupAnima.fillMode = kCAFillModeForwards;
    groupAnima.removedOnCompletion = NO;
    
    [self.iconView.layer addAnimation:groupAnima forKey:nil];
}
```

## UIView封装动画
### UIView动画（首尾）
#### 简介
执行动画所需要的工作由UIView类自动完成，但仍要在希望执行动画时通知视图，为此需要将改变属性的代码放在[UIView beginAnimations:nil context:nil]和[UIView commitAnimations]之间。
常见方法：
- **+ (void)setAnimationDelegate:(id)delegate**     设置动画代理对象，当动画开始或者结束时会发消息给代理对象
- **+ (void)setAnimationWillStartSelector:(SEL)selector**   当动画即将开始时，执行delegate对象的selector，并且把beginAnimations:context:中传入的参数传进selector
- **+ (void)setAnimationDidStopSelector:(SEL)selector**  当动画结束时，执行delegate对象的selector，并且把beginAnimations:context:中传入的参数传进selector
- **+ (void)setAnimationDuration:(NSTimeInterval)duration**   动画的持续时间，秒为单位
- **+ (void)setAnimationDelay:(NSTimeInterval)delay**  动画延迟delay秒后再开始
- **+ (void)setAnimationStartDate:(NSDate \*)startDate**   动画的开始时间，默认为now
- **+ (void)setAnimationCurve:(UIViewAnimationCurve)curve**  动画的节奏控制
- **+ (void)setAnimationRepeatCount:(float)repeatCount**  动画的重复次数
- **+ (void)setAnimationRepeatAutoreverses:(BOOL)repeatAutoreverses**  如果设置为YES,代表动画每次重复执行的效果会跟上一次相反
- **+ (void)setAnimationTransition:(UIViewAnimationTransition)transition forView:(UIView \*)view cache:(BOOL)cache**  设置视图view的过渡效果, transition指定过渡类型, cache设置YES代表使用视图缓存，性能较好

#### 示例
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //打印动画块的位置
    NSLog(@"动画执行之前的位置：%@",NSStringFromCGPoint(self.customView.center));
    
    //首尾式动画
    [UIView beginAnimations:nil context:nil];
    //执行动画
    //设置动画执行时间
    [UIView setAnimationDuration:2.0];
    //设置代理
    [UIView setAnimationDelegate:self];
    //设置动画执行完毕调用的事件
    [UIView setAnimationDidStopSelector:@selector(didStopAnimation)];
    self.customView.center=CGPointMake(200, 300);
    [UIView commitAnimations];

}

-(void)didStopAnimation
{
    NSLog(@"动画执行完毕");
    //打印动画块的位置
    NSLog(@"动画执行之后的位置：%@",NSStringFromCGPoint(self.customView.center));
}
```

#### UIView封装的动画与CALayer动画的对比
使用UIView和CALayer都能实现动画效果，但是在真实的开发中，一般还是主要使用UIView封装的动画，而很少使用CALayer的动画。

**CALayer核心动画与UIView动画的区别**：
UIView封装的动画执行完毕之后不会反弹。即如果是通过CALayer核心动画改变layer的位置状态，表面上看虽然已经改变了，但是实际上它的位置是没有改变的。

### block动画
#### 方法：
- **+(void)animateWithDuration:delay:options:animations:completion:**
- **+(void)transitionWithView:duration:options:animations:completion:**
- **+(void)transitionFromView:toView:duration:options:completion:**
属性简介：
1. duration：动画的持续时间
2. delay：动画延迟delay秒后开始
3. options：动画的节奏控制/转场动画的类型(重复，转场等)
4. animations：将改变视图属性的代码放在这个block中
5. completion：动画结束后，会自动调用这个block

前两个方法用起来好像没什么区别。

#### 示例：
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //block代码块动画
        [UIView transitionWithView:self.customView duration:3.0 options:0 animations:^{
            //执行的动画
            NSLog(@"动画开始执行前的位置：%@",NSStringFromCGPoint(self.customView.center));
            self.customView.center=CGPointMake(200, 300);
        } completion:^(BOOL finished) {
            //动画执行完毕后的首位操作
            NSLog(@"动画执行完毕");
            NSLog(@"动画执行完毕后的位置：%@",NSStringFromCGPoint( self.customView.center));
        }];
}
```


