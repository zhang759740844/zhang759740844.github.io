title: 移位与枚举
date: 2016/9/7 10:07:12  
categories: IOS
tags:
	- Objective-C

---

oc中的枚举使用enum，iOS本身定义的枚举里面经常会使用左移（<<）来定义枚举的值，其原因是什么呢？

<!--more-->
## enum的基本用法使用
定义一组常量。
```objc
typedef enum{
	BlenderSpeedStir=1，
	BlenderSpeedChop=1<<1，
} BlenderSpeed；
```

还可以写成这样：
```objc
typedef NS_ENUM(NSInteger, BlenderSpeed) {
//以下是枚举成员
    Test1A = 1,
    Test1B = 1<<1,
    Test1C = 1<<2,
    Test1D = 1<<3
};
```
使用： 
```
BlenderSpeed speed = Test1A；
```
使用`BlenderSpeed`代替`NSInteger`。


## 移位定义enum
### 使用移位定义enum的好处
enum一般的枚举数值都用移位表示，一般和数字没有太大差别。不过，比起使用数字的一个显著的好处就是：**用移位来定义枚举能把1的位置错开，可以使用`按位或`将几个枚举值表示成一个数，而不会互相影响**。如果用数字就要使用一个数组。

比如：
```objc
[UIView animateWithDuration:1 delay:0 options:UIViewAnimationOptionTransitionFlipFromRight |UIViewAnimationOptionRepeat animations:^{
	nil
} completion:^(BOOL finished) {
	nil;
}];
```

其中`options`只能传入一个数值，使用数字就不能传入多个枚举值，但是当enum使用了移位，那么久可以传入`UIViewAnimationOptionTransitionFlipFromRight |UIViewAnimationOptionRepeat`表示既`FlipFromRight`又`repeat`。

### 取出enum值
将几个枚举值放入一个数中，方法处理的时候总的取出来吧。那么怎么取呢？先来看一下`按位与&`操作符的特性：

>数b与已知数a进行按位与操作。a中为0的位，结果必为0，a中为1的位，结果为b中相应的位。
所以，若要判断某一个数中的某一位是否为1，只要将这个数与一个该位为1其余位为0的数进行按位与，结果不为0表示该位为1，反之为0。

因此，我们可以这样取出枚举值：
```objc
typedef enum{
    a = 1 << 0,
    b = 1 << 1,
    c = 1 << 2,
    d = 1 << 3
}testEnum;

testEnum e = a | b;

if (e & a) {
    printf("满足条件a");
    //满足a要做的事
}
if (e & b) {
    printf("满足条件b");
    //满足b要做的事
}
if (e & c) {
    printf("满足条件c");
    //满足c要做的事
}
```
将数值与各个枚举值按位与，非零即包含该枚举值。

也可以通过一个for循环，再通过switch-case处理：
```objc
typedef NS_ENUM(NSInteger, AnimationType) {
    //以下是枚举成员
    BaseAnimation = 1,
    KeyFrameAnimation_Value = 1<<1,
    KeyFrameAnimation_Path = 1<<2,
    KeyFrameAnimation_shake = 1<<3,
    Transition = 1<<4,
    AnimationGroup = 1<<5,
    ViewAnimation = 1<<6
};

- (void)initAnimationWithType:(NSInteger)type{
    for (NSInteger animationType = 1; animationType<(ViewAnimation<<1); animationType = animationType<<1) {
        if (type & animationType) {
            [self startAnimationInitialWithType:animationType];
        }
    }
}

- (void)startAnimationInitialWithType:(NSInteger)type{
    
    switch (type) {
        case BaseAnimation:{
            CALayer *myLayer=[CALayer layer];
            myLayer.bounds=CGRectMake(0, 0, 50, 80);
            myLayer.backgroundColor=[UIColor yellowColor].CGColor;
            myLayer.position=CGPointMake(50, 50);
            myLayer.anchorPoint=CGPointMake(0, 0);
            myLayer.cornerRadius=20;
            //添加layer
            [self.view.layer addSublayer:myLayer];
            self.myLayer=myLayer;
        }
            break;
        case Transition:
            _index = 1;
            break;
        default:
            break;
    }
}
```

**需要注意的是**：`case`中如果声明了变量，必须要用`{}`包住，这是编译器强制的，不然会报错，应该是为了避免[switch case语句里面不能定义对象，有语法错误，除非加一个花括号](http://blog.csdn.net/fanjunxi1990/article/details/9162945)这个问题。

### 引申
上面是使用了2进制来错开，保留每个位，其实其他进制也可以,但位数是2的n次方。
比如0000 0000 8个位，可以前4个位存储一个值，后4个位存储一个值：
```objc

typedef enum{
    a = 0 << 0,
    b = 1 << 0,
    c = 2 << 0,
    d = 3 << 0,

    e = 0 << 4,
    f = 1 << 4,
    g = 2 << 4,
    h = 3 << 4
}testEnum;
```

a b c d的前4为都是0，值的变化在后4位，而e f g h正好相反。a,b,c,d之间无法通过按位与区分，但是a,b,c,d可以和e,f,g,h区分。这样，整个枚举值被分成了两组。如何判断值是哪个组的枚举值呢？很简单，和`00001111`按位与，后四位中有一个数为1，那么整个按位与的值就不为0。同理，与`11110000`按位与得到另外一组。

>Demo详见CALayer_Transform
