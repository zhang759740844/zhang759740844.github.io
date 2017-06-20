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

































 