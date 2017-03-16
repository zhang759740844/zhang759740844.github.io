title: NSLayoutConstraint 学习笔记
date: 2016/9/8 10:07:12  
categories: iOS
tags:
	- UI
---

`AutoLayout`是一种基于约束的，描述性的布局系统。参考了网上的许多文章，进行总结。基本还是以ios6为主，ios8的一些特性并没有仔细研究。以后需要用到再说。
参考与[AutoLayout详解](http://www.jianshu.com/p/79f92139ffdf)

<!--more-->

## 代码添加AutoLayout
### 关闭Autoresizing
`AutoLayout`旨在替代`Autoresizing`，所以在同一个项目中，`AutoLayout`和`Autoresizing`是不能共存的。要实现自动布局，必须关掉view的`AutoresizeingMask`

```objc
view.translatesAutoresizingMaskIntoConstraints = NO;
```
这里的view指的是需要添加约束的view。

### NSLayoutConstraint相关
#### 约束方法
```objc
+(instancetype)constraintWithItem:(id)view1                  //被约束的视图一
                        attribute:(NSLayoutAttribute)attr1   //view1的属性
                        relatedBy:(NSLayoutRelation)relation //左右视图的关系
                           toItem:(id)view2                  //被约束的视图二
                        attribute:(NSLayoutAttribute)attr2   //view2的属性
                       multiplier:(CGFloat)multiplier        //乘数
                         constant:(CGFloat)c;                //常量
```

公式是这样的：`view1.attr1 = view2.attr2 * multiplier + constant`

#### NSLayoutAttribute属性
```objc
typedef NS_ENUM(NSInteger, NSLayoutAttribute) {
    NSLayoutAttributeLeft = 1, //左边
    NSLayoutAttributeRight,    //右边
    NSLayoutAttributeTop,      //顶部
    NSLayoutAttributeBottom,   //底部
    NSLayoutAttributeLeading,  //首部
    NSLayoutAttributeTrailing, //尾部
    NSLayoutAttributeWidth,    //宽度
    NSLayoutAttributeHeight,   //高度
    NSLayoutAttributeCenterX,  //X轴中心
    NSLayoutAttributeCenterY,  //Y轴中心
    NSLayoutAttributeBaseline, //基线
    NSLayoutAttributeLastBaseline = NSLayoutAttributeBaseline,
    NSLayoutAttributeFirstBaseline NS_ENUM_AVAILABLE_IOS(8_0),

    //iOS8的暂时没有研究
    NSLayoutAttributeLeftMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeRightMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTopMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeBottomMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeLeadingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTrailingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterXWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterYWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),

    NSLayoutAttributeNotAnAttribute = 0 //无属性
};
```

#### NSLayoutRelation关系
```objc
typedef NS_ENUM(NSInteger, NSLayoutRelation) {
    NSLayoutRelationLessThanOrEqual = -1,   //小于等于
    NSLayoutRelationEqual = 0,              //等于
    NSLayoutRelationGreaterThanOrEqual = 1, //大于等于
};
```

### 创建view
```objc
//创建view
UIView *view = [[UIView alloc] init]; //这里不需要设置frame
view.backgroundColor = [UIColor brownColor];
view.translatesAutoresizingMaskIntoConstraints = NO; //要实现自动布局，必须把该属性设置为NO
[self.view addSubview:view];

//添加约束
[self.view addConstraint:
 [NSLayoutConstraint constraintWithItem:view
                              attribute:NSLayoutAttributeLeft
                              relatedBy:NSLayoutRelationEqual
                                 toItem:self.view
                              attribute:NSLayoutAttributeLeft
                             multiplier:1
                               constant:20]];
[self.view addConstraint:
 [NSLayoutConstraint constraintWithItem:view
                              attribute:NSLayoutAttributeRight
                              relatedBy:NSLayoutRelationEqual
                                 toItem:self.view
                              attribute:NSLayoutAttributeRight
                             multiplier:1
                               constant:-10]];
[self.view addConstraint:
 [NSLayoutConstraint constraintWithItem:view
                              attribute:NSLayoutAttributeTop
                              relatedBy:NSLayoutRelationEqual
                                 toItem:self.view
                              attribute:NSLayoutAttributeTop
                             multiplier:1
                               constant:30]];
[self.view addConstraint:
 [NSLayoutConstraint constraintWithItem:view
                              attribute:NSLayoutAttributeBottom
                              relatedBy:NSLayoutRelationEqual
                                 toItem:self.view
                              attribute:NSLayoutAttributeBottom
                             multiplier:1
                               constant:-20]];
```

### 添加约束的规则
- 对于两个同层级view之间的约束关系，添加到它们的父view上
- 对于两个不同层级view之间的约束关系，添加到他们最近的共同父view上
- 对于有层次关系的两个view之间的约束关系，添加到层次较高的父view上

### Autolayout来实现动画功能
在执行动画时记得调用一下方法：
```objc
//在修改了约束之后，只要执行下面代码，就能做动画效果
[UIView animateWithDuration:0.5 animations:^{
      [self.view layoutIfNeeded];
}];
```

## 第三方框架实现AutoLayout
Masonry框架是目前最流行的Autolayout第三方框架，用优雅的代码方式编写Autolayout，省去了苹果官方恶心的Autolayout代码，大大提高了开发效率。
### 框架导入
这里先介绍下ios的包管理工具cocoaPods，如何下载安装就不说了，介绍下如何使用。
#### 创建Podfile
在工程的目录下新建Podfile文件，写入我们需要的第三方库：
```ruby
target 'NSLayoutConstraintDemo' do
pod 'Masonry'
end
```

当然，这其实不用自己设置，通过终端进入工程目录，输入：

```ruby
pod init
```

即可初始化完成一个 Podfile

#### 导入第三方库

进入终端，运行命令`pod install`,导入第三方库。

### Masonry基本使用
最基本的使用方式：
```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.属性.equalTo(另一个view).with.insets(差值);
}];
```

### Masonry属性
```objc
@property (nonatomic, strong, readonly) MASConstraint *left;
@property (nonatomic, strong, readonly) MASConstraint *top;
@property (nonatomic, strong, readonly) MASConstraint *right;
@property (nonatomic, strong, readonly) MASConstraint *bottom;
@property (nonatomic, strong, readonly) MASConstraint *leading;
@property (nonatomic, strong, readonly) MASConstraint *trailing;
@property (nonatomic, strong, readonly) MASConstraint *width;
@property (nonatomic, strong, readonly) MASConstraint *height;
@property (nonatomic, strong, readonly) MASConstraint *centerX;
@property (nonatomic, strong, readonly) MASConstraint *centerY;
@property (nonatomic, strong, readonly) MASConstraint *baseline;
```
其中leading与left trailing与right 在正常情况下是等价的。
### 使用实例
居中显示一个view：
```objc
//从此以后基本可以抛弃CGRectMake了
UIView *sv = [UIView new];

//在做autoLayout之前 一定要先将view添加到superview上 否则会报错
[self.view addSubview:sv];

//mas_makeConstraints就是Masonry的autolayout添加函数 将所需的约束添加到block中行了
[sv mas_makeConstraints:^(MASConstraintMaker *make) {

    //将sv居中(很容易理解吧?)
    make.center.equalTo(self.view);
    
    //将size设置成(300,300)
    make.size.mas_equalTo(CGSizeMake(300, 300));
}];
```

### 方法简析
#### 添加autolayout约束
有三个可以添加约束的方法：
```objc
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *make))block;
- (NSArray *)mas_updateConstraints:(void(^)(MASConstraintMaker *make))block;
- (NSArray *)mas_remakeConstraints:(void(^)(MASConstraintMaker *make))block;

/*
    mas_makeConstraints 只负责新增约束 Autolayout不能同时存在两条针对于同一对象的约束 否则会报错 
    mas_updateConstraints 针对上面的情况 会更新在block中出现的约束 不会导致出现两个相同约束的情况
    mas_remakeConstraints 则会清除之前的所有约束 仅保留最新的约束
    
    三种函数善加利用 就可以应对各种情况了
*/
```

#### equalTo 和 mas_equalTo的区别
`mas_equalTo`只是对其参数进行了一个BOX操作(装箱)。所支持的类型 除了`NSNumber`支持的那些数值类型之外 就只支持`CGPoint` `CGSize` `UIEdgeInsets`

### 另一个示例
```objc
UIView *sv1 = [UIView new];
[sv1 showPlaceHolder];
sv1.backgroundColor = [UIColor redColor];
[sv addSubview:sv1];
[sv1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(sv).with.insets(UIEdgeInsetsMake(10, 10, 10, 10));
    
    /* 等价于
    make.top.equalTo(sv).with.offset(10);
    make.left.equalTo(sv).with.offset(10);
    make.bottom.equalTo(sv).with.offset(-10);
    make.right.equalTo(sv).with.offset(-10);
    */
    
    /* 也等价于
    make.top.left.bottom.and.right.equalTo(sv).with.insets(UIEdgeInsetsMake(10, 10, 10, 10));
    */
}];
```
可以看到 edges 其实就是top,left,bottom,right的一个简化 分开写也可以 一句话更省事.

这里and和with 两个函数什么事情都没做.

### 还有一个示例
```objc
UIView *lastView = nil;
    
for ( int i = 1 ; i <= count ; ++i ){
    UIView *subv = [UIView new];
    [container addSubview:subv];
    subv.backgroundColor = [UIColor colorWithHue:( arc4random() % 256 / 256.0 )
                                      saturation:( arc4random() % 128 / 256.0 ) + 0.5
                                      brightness:( arc4random() % 128 / 256.0 ) + 0.5
                                           alpha:1];
        
    [subv mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.and.right.equalTo(container);
        make.height.mas_equalTo(@(20*i));
           
        if ( lastView ){
            make.top.mas_equalTo(lastView.mas_bottom);
        }
        else{
            make.top.mas_equalTo(container.mas_top);
        }
    }];
       
    lastView = subv;
}
   
    
[container mas_makeConstraints:^(MASConstraintMaker *make) {
    make.bottom.equalTo(lastView.mas_bottom);
}];
```

框架通过范畴为UIView添加了`mas_bottom`、`mas_top`等属性，用来表示view的上下左右的位置。这样，有利于**在view之间的约束条件的建立**，之前的示例都是view与父view的约束关系。

还有一些其他的示例，可以参见[Masonry介绍与使用实践](http://adad184.com/2014/09/28/use-masonry-to-quick-solve-autolayout/)

>Demo 详见NSLayoutConstraintDemo





