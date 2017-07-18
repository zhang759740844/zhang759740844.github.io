title: Masonry 源码解析
date: 2017/6/19 10:07:12  
categories: iOS
tags:
	- Xcode
---

Masonry 是 iOS 中的一套布局框架，先占个坑学习下源码。

<!--more-->

## 链式编程
Masonry 的一大特点就是链式编程。作为预备知识，先来观察一下链式编程的特点：
```objc
[View mas_makeConstraints:^(MASConstraintMaker *make) {
      make.top.bottom.left.right.equalTo(anotherView);
}];
```
我们知道`.`在 oc 中就是调用该对象相应属性的 get 方法，所以 `top` 其实就是 `make` 的一个属性：

```objc
@property (nonatomic, strong, readonly) MASConstraint *top;
```

那么这个 get 方法要返回什么呢？由于后面接着要调用 `.bottom`，因此 `make.top` 返回的应该也是一个 `make` 对象，即 self本身:

```objc
- (MASConstraint *)top {
	....
}
```



### 一个耳熟能详的计算器例子

之所以说耳熟能详是因为基本每个讲链式编程的文章都会用到这个例子。我也简单粘贴一下代码：

```objc
// NSObject+CaculatorMaker.h
@class CaculatorMaker;
@interface NSObject (CaculatorMaker)

//计算
+ (int)makeCaculators:(void(^)(CaculatorMaker *make))caculatorMaker;

@end


// CaculatorMaker.h
@interface CaculatorMaker : NSObject

@property (nonatomic, assign) int iResult;

//加法
- (CaculatorMaker *(^)(int))add;

@end
```

其中 `CaculatorMaker` 就类似于上面的 `MASConstraint`,把计算结果 `iResult` 保存在了里面。

和上面不同的是，上面的 `top` 返回的类型直接就是 `MASConstraint`,而下面这个返回的是一个返回类型为 `CaculatorMaker` 的 block。这是因为，上面是没有入参的get方法，而这个计算器的加减乘除是需要入参的。比如 `a.add(1)` 这样的调用，会被分解为 `a.add` 和 `(1)`，只有 `a.add` 返回的是个 block 才能后续调用 `(1)` 处理。

```objc
// NSObject+CaculatorMaker.m
#import "NSObject+CaculatorMaker.h"
#import "CaculatorMaker.h"

@implementation NSObject (CaculatorMaker)

//计算
+ (int)makeCaculators:(void(^)(CaculatorMaker *make))block
{
    CaculatorMaker *mgr = [[CaculatorMaker alloc] init];
    block(mgr);
    return mgr.iResult;
}

@end
  
// CaculatorMaker.m
#import "CaculatorMaker.h"

@implementation CaculatorMaker

- (CaculatorMaker *(^)(int))add
{
   return ^(int value)
    {
        _iResult += value;
        return self;
    };
}

@end
```

调用方式：

```objc
int iResult = [NSObject makeCaculators:^(CaculatorMaker *make) {
     make.add(1).add(2).add(3).divide(2);
}];
```

`make.add(1).add(2).add(3).divide(2);` 中就包含着要返回的值了，但是因为要直接返回一个 `iResult` 而不是返回一个 `CaculatorMaker` 对象，所以给 `NSObject` 提供一个方法，将要处理的代码通过 block 传入。



## 使用

### 使用详解

1. `mas_makeConstraints` 是给view添加约束，约束有几种，分别是边距，宽，高，左上右下距离，基准线。添加过约束后可以有修正，修正 有`offset`（位移）修正和 `multipliedBy`（倍率）修正。（倍率修正非常好用，可以等分父view）
2. 使用 `mas_makeConstraints` 方法的元素必须事先添加到父元素的中，例如 `[self.view addSubview:view];`
3. `mas_equalTo` 和 `equalTo` 区别：`mas_equalTo` 比`equalTo`多了类型转换操作，一般来说，大多数时候两个方法都是 通用的，但是对于数值元素使用 `mas_equalTo`。对于对象或是多个属性的处理，使用`equalTo`。特别是多个属性时，必须使用 `equalTo`,例如 `make.left.and.right.equalTo(self.view);`
4. 注意到方法 `with` 和 `and`,这连个方法其实没有做任何操作，方法只是返回对象本身，这这个方法的左右完全是为了方法写的时候的可读性 。`make.left.and.right.equalTo(self.view);` 和`make.left.right.equalTo(self.view);` 是完全一样的，但是明显的加了 `and` 方法的语句可读性 更好点。

### 例子

#### 初级例子：

```objc
// exp1: 中心点与self.view相同，宽度为400*400
-(void)exp1{

    UIView *view = [UIView new];
    [view setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:view];
    [view mas_makeConstraints:^(MASConstraintMaker *make) {

         make.center.equalTo(self.view);
         make.size.mas_equalTo(CGSizeMake(400,400));
    }];

}


//exp2: 上下左右边距都为10
-(void)exp2{

    UIView *view = [UIView new];
    [view setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:view];
    [view mas_makeConstraints:^(MASConstraintMaker *make) {

        make.edges.equalTo(self.view).with.insets(UIEdgeInsetsMake(10, 10, 10, 10));

        //  make.left.equalTo(self.view).with.offset(10);
        //  make.right.equalTo(self.view).with.offset(-10);
        //  make.top.equalTo(self.view).with.offset(10);
        //  make.bottom.equalTo(self.view).with.offset(-10);
    }];

}

//exp3 让两个高度为150的view垂直居中且等宽且等间隔排列 间隔为10
-(void)exp3{

    UIView *view1 = [UIView new];
    [view1 setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:view1];

    UIView *view2 = [UIView new];
    [view2 setBackgroundColor:[UIColor redColor]];
    [self.view addSubview:view2];

    [view1 mas_makeConstraints:^(MASConstraintMaker *make) {

        make.centerY.mas_equalTo(self.view.mas_centerY);
        make.height.mas_equalTo(150);
        make.width.mas_equalTo(view2.mas_width);
        make.left.mas_equalTo(self.view.mas_left).with.offset(10);
        make.right.mas_equalTo(view2.mas_left).offset(-10);

    }];
    [view2 mas_makeConstraints:^(MASConstraintMaker *make) {

        make.centerY.mas_equalTo(self.view.mas_centerY);
        make.height.mas_equalTo(150);
        make.width.mas_equalTo(view1.mas_width);
        make.left.mas_equalTo(view1.mas_right).with.offset(10);
        make.right.equalTo(self.view.mas_right).offset(-10);

    }];

}
```

#### 计算器布局

效果同iOS 自带计算器：

```objc
//高级布局练习 iOS自带计算器布局
-(void)exp4{


    //申明区域，displayView是显示区域，keyboardView是键盘区域
    UIView *displayView = [UIView new];
    [displayView setBackgroundColor:[UIColor blackColor]];
    [self.view addSubview:displayView];

    UIView *keyboardView = [UIView new];
    [self.view addSubview:keyboardView];

    //先按1：3分割 displView（显示结果区域）和 keyboardView（键盘区域）
    [displayView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.view.mas_top);
        make.left.and.right.equalTo(self.view);
        make.height.equalTo(keyboardView).multipliedBy(0.3f);
    }];

    [keyboardView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(displayView.mas_bottom);
        make.bottom.equalTo(self.view.mas_bottom);
        make.left.and.right.equalTo(self.view);

    }];

    //设置显示位置的数字为0
    UILabel *displayNum = [[UILabel alloc]init];
    [displayView addSubview:displayNum];
    displayNum.text = @"0";
    displayNum.font = [UIFont fontWithName:@"HeiTi SC" size:70];
    displayNum.textColor = [UIColor whiteColor];
    displayNum.textAlignment = NSTextAlignmentRight;
    [displayNum mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.and.right.equalTo(displayView).with.offset(-10);
        make.bottom.equalTo(displayView).with.offset(-10);
    }];


    //定义键盘键名称，？号代表合并的单元格
    NSArray *keys = @[@"AC",@"+/-",@"%",@"÷"
                     ,@"7",@"8",@"9",@"x"
                     ,@"4",@"5",@"6",@"-"
                     ,@"1",@"2",@"3",@"+"
                     ,@"0",@"?",@".",@"="];


    int indexOfKeys = 0;
    for (NSString *key in keys){
        //循环所有键
        indexOfKeys++;
        int rowNum = indexOfKeys %4 ==0? indexOfKeys/4:indexOfKeys/4 +1;
        int colNum = indexOfKeys %4 ==0? 4 :indexOfKeys %4;
        NSLog(@"index is:%d and row:%d,col:%d",indexOfKeys,rowNum,colNum);

        //键样式
        UIButton *keyView = [UIButton buttonWithType:UIButtonTypeCustom];
        [keyboardView addSubview:keyView];
        [keyView setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
        [keyView setTitle:key forState:UIControlStateNormal];
        [keyView.layer setBorderWidth:1];
        [keyView.layer setBorderColor:[[UIColor blackColor]CGColor]];
        [keyView.titleLabel setFont:[UIFont fontWithName:@"Arial-BoldItalicMT" size:30]];

        //键约束
        [keyView mas_makeConstraints:^(MASConstraintMaker *make) {

            //处理 0 合并单元格
            if([key isEqualToString:@"0"] || [key isEqualToString:@"?"] ){

                if([key isEqualToString:@"0"]){
                    [keyView mas_makeConstraints:^(MASConstraintMaker *make) {
                        make.height.equalTo(keyboardView.mas_height).with.multipliedBy(.2f);
                        make.width.equalTo(keyboardView.mas_width).multipliedBy(.5);
                        make.left.equalTo(keyboardView.mas_left);
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.9f);
                    }];
                }if([key isEqualToString:@"?"]){
                    [keyView removeFromSuperview];
                }

            }
            //正常的单元格
            else{
                make.width.equalTo(keyboardView.mas_width).with.multipliedBy(.25f);
                make.height.equalTo(keyboardView.mas_height).with.multipliedBy(.2f);

                //按照行和列添加约束，这里添加行约束
                switch (rowNum) {
                    case 1:
                    {
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.1f);
                        keyView.backgroundColor = [UIColor colorWithRed:205 green:205 blue:205 alpha:1];

                    }
                        break;
                    case 2:
                    {
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.3f);
                    }
                        break;
                    case 3:
                    {
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.5f);
                    }
                        break;
                    case 4:
                    {
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.7f);
                    }
                        break;
                    case 5:
                    {
                        make.baseline.equalTo(keyboardView.mas_baseline).with.multipliedBy(.9f);
                    }
                        break;
                    default:
                        break;
                }
                //按照行和列添加约束，这里添加列约束
                switch (colNum) {
                    case 1:
                    {
                        make.left.equalTo(keyboardView.mas_left);

                    }
                        break;
                    case 2:
                    {
                        make.right.equalTo(keyboardView.mas_centerX);

                    }
                        break;
                    case 3:
                    {
                        make.left.equalTo(keyboardView.mas_centerX);
                    }
                        break;
                    case 4:
                    {
                        make.right.equalTo(keyboardView.mas_right);
                        [keyView setBackgroundColor:[UIColor colorWithRed:243 green:127 blue:38 alpha:1]];
                    }
                        break;
                    default:
                        break;
                }
            }
        }];
    }

}
```

本例子使用的 `baseline` 去控制高度位置，这似乎不是太准，如果想要精准控制高度位置，可以使用一行一行添加的方法，每次当前行的 top 去 `equelTo` 上一行的 bottom。

```objc
for（遍历所有行)
    for（遍历所以列）
    //当前行约束根据上一行去设置
    ......
```

## 源码解析































 