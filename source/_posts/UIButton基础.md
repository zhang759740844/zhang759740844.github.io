title: UIButton 简介
date: 2016/8/29 14:07:12  
categories: iOS
tags: 
	- 基本控件
	
---

从最基础的控件开始一点点学习。先来总结下UIButton。

<!--more-->

## 基础
### 创建
```objc
UIButton *btn1 = [[UIButton alloc] init];
CGRect btn1Frame = CGRectMake(50, 50, 200, 100);
btn1.frame = btn1Frame;
[self.view addSubview:btn1];
```

通过`init`和设置`frame`以及`addSubView`就可以将button添加到Voew上。

### 点击事件
#### 通过xib关联
```objc
- (IBAction)btn2Pressed:(id)sender{
    NSLog(@"点击操作");
}
```
与xib关联即可，其中`(id)sender`就表示的是这个Button

#### 通过代码
```objc
//添加
[btn1 addTarget:self action:@selector(btn1Pressed:) forControlEvents:UIControlEventTouchUpInside];
//移除
[btn1 removeTarget:self action:@selector(btn1Pressed:) forControlEvents:UIControlEventTouchUpInside];
```
为`btn1`添加和删除一个`btn1Pressed`的点击事件。

## 设置title和image
button有`imageView`和`titleLabel`两个属性，默认image在左，label在右。

### 添加title和image
```
[btn1 setImage:[UIImage imageNamed:@"Image"] forState:UIControlStateNormal];
[btn1 setTitle:@"BTN1" forState:UIControlStateNormal];
```
这里`forState`常用的有一下几种：
- UIControlStateNormal  		常态
- UIControlStateHighlighted 	高亮
- UIControlStateDisabled		禁用
- UIControlStateSelected		选中

其中需要说明的是，高亮就是点击时的状态。其实还有一种`UIControlStateSelected | UIControlStateHighlighted`这个组合是选中时候的高亮状态，也是比较有用的。

### 设置button选中
button选中与否是由`UIControlStateSelected`控制的。

可能会遇到这样的情景：点击一下button切换成另外一个图，再点一下切换回去，实际上就是类似于*点赞*。一般情况我们实现方式是这样的：
```objc
- (void)buttonClick:(UIButton *)button {
    if ([button.currentImage isEqual:[UIImage imageNamed:@"like"]]) {
        [button setImage:[UIImage imageNamed:@"like_selected"] forState:UIControlStateNormal];
    }
    else {
        [button setImage:[UIImage imageNamed:@"like"] forState:UIControlStateNormal];
    }
}
```

但是最好不要这样实现。可以使用`UIControlStateSelected`来控制：
```objc
[btn1 setImage:[UIImage imageNamed:@"Image"] forState:UIControlStateNormal];
[btn1 setImage:[UIImage imageNamed:@"Image2"] forState:UIControlStateSelected];

//点击事件
- (void)btn1Pressed:(UIButton *)button{
    //button.enabled 设置是否可点击。
    button.selected = !button.selected;
}
```
这样，每次点击的时候切换选中状态达到效果。

但是，这样处理会遇到这样一个问题:不管按钮从normal状态转为selected状态,还是反过来,中间都会经历一个highLighted状态,这就导致在状态切换的过程中有一次图片的跳变。

因此，需要将normal->selected以及selected->normal之间的highlight都设置一下:
```objc
//normal->selected
[btn1 setImage:[UIImage imageNamed:@"Image"] forState:UIControlStateHighlighted];
//selected->normal
[btn1 setImage:[UIImage imageNamed:@"Image2"] forState:UIControlStateSelected | UIControlStateHighlighted];
```

好的，这样就完成了切换状态的过程。

### 设置image和title位置
image和title默认image在左，title紧贴在其右边。不过这个位置其实是可以改变的。

#### image和title位置互换
先看代码：
```objc
[btn1 setTitleEdgeInsets:UIEdgeInsetsMake(0, -btn1.imageView.bounds.size.width, 0, btn1.imageView.bounds.size.width)];
[btn1 setImageEdgeInsets:UIEdgeInsetsMake(0, btn1.titleLabel.bounds.size.width, 0, -btn1.titleLabel.bounds.size.width)];
```
这里需要强调的是一定要先设置`title`再设置`image`。因为`title`是依赖于`image`的，在如果先设置`image`，这个时候`title`还没有确定，于是`titleLabel.bounds.size.width`一定是`0`，即`image`没有变。

这里的`UIEdgeInsetsMake`里的四个参数分别是`top`,`left`,`bottom`,`right`四个方向的`inset`,默认是`0`，也就是说，所有变化都是针对当前位置的。`-btn1.imageView.bounds.size.width`表示让`title`的左边距**减少**`image`的宽度，同理`btn1.imageView.bounds.size.width`表示让右边距**增加**`image`的宽度。

#### 控制image的大小
交换了image的位置后，我就想怎么控制image的大小。尝试改变了`UIEdgeInsetsMake`的参数，发现改变`top`和`bottom`可以将图片压缩，但是改变`left` `right`图片始终不动。

经过我不断尝试后终于得出了结论：
以横轴为例，只有当**左边距+右边距+图片宽度=button宽度**时，继续增加边距，才会导致图片的压缩。当**左边距+右边距+图片宽度<button宽度**时，增加一边会让图像像另一边移动；同时增加两边，两边抵消，在原处不动。

因此，为什么改变`top`和`bottom`可以将图片压缩呢？因为由于button高度较小，image在纵轴将button填满，此时`top`和`bottom`都为0，但是由于**image高度=button高度**，此时增加`top`和`bottom`就会将图片压缩。当然，如果一个增加，一个等量减小，图片就会像减少那边移动。

而在横轴方面，开始时，**左边距+右边距+图片宽度<button宽度**。图片只会平移直到边距增加到使等式相等才会进行压缩。

## UIControl
### 概览
`UIControl` 是控件类的基类，它是一个抽象基类，我们不能直接使用 `UIControl` 类来实例化控件，它只是为控件子类定义一些通用的接口，并提供一些基础实现，以在事件发生时，预处理这些消息并将它们发送到指定目标对象上。
![UIControl](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/uibutton_1.png?raw=true)

### Target-Action机制
Target-action 是一种设计模式，直译过来就是”目标-行为”。当我们通过代码为一个按钮添加一个点击事件时，通常是如下处理：

```objc
[button addTarget:self action:@selector(tapButton:) forControlEvents:UIControlEventTouchUpInside];
```

也就是说，当按钮的点击事件发生时，会将消息发送到 `target`(此处即为 `self` 对象)，并由 `target` 对象的 `tapButton:` 方法来处理相应的事件。因此，Target-Action 机制由两部分组成：即目标对象和行为 `Selector`。目标对象指定最终处理事件的对象，而行为 `Selector` 则是处理事件的方法。

我们先来看看 UIControl 为我们提供了哪些自定义跟踪行为的方法:
```objc
- (BOOL)beginTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event
- (BOOL)continueTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event
- (void)endTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event
- (void)cancelTrackingWithEvent:(UIEvent *)event
```

这四个方法分别对应的时跟踪开始、移动、结束、取消四种状态。跟 `UIResponse` 提供的四个事件跟踪方法是不是挺像的？我们来看看 `UIResponse` 的四个方法：

```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event
```

上面两组方法的参数基本相同，只不过 `UIControl` 的是针对单点触摸，而 `UIResponse` 可能是多点触摸。另外，返回值也是大同小异。由于 `UIControl` 本身是视图，所以它实际上也继承了 `UIResponse` 的这四个方法。如果测试一下，我们会发现**在针对控件的触摸事件发生时，这两组方法都会被调用，而且互不干涉。**

对于一个给定的事件，`UIControl` 会调用 `sendAction:to:forEvent:` 来将行为消息转发到 `UIApplication`对象，再由 `UIApplication` 对象调用其 `sendAction:to:fromSender:forEvent:` 方法来将消息分发到指定的 `target` 上。而如果子类想监控或修改这种行为的话，则可以重写这个方法:

```objc
// ImageControl.m
- (void)sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
  // 将事件传递到对象本身来处理
    [super sendAction:@selector(handleAction:) to:self forEvent:event];
}
 
- (void)handleAction:(id)sender {
 
    NSLog(@"handle Action");
}
 
// ViewController.m
 
- (void)viewDidLoad {
    [super viewDidLoad];
 
    self.view.backgroundColor = [UIColor whiteColor];
 
    ImageControl *control = [[ImageControl alloc] initWithFrame:(CGRect){50.0f, 100.0f, 200.0f, 300.0f} title:@"This is a demo" image:[UIImage imageNamed:@"demo"]];
    // ...
 
    [control addTarget:self action:@selector(tapImageControl:) forControlEvents:UIControlEventTouchUpInside];
}
- (void)tapImageControl:(id)sender {
 
    NSLog(@"sender = %@", sender);
}
```

由于我们重写了 `sendAction:to:forEvent:` 方法，所以最后处理事件的 `Selector` 是 `ImageControl的handleAction:` 方法，而不是 `ViewController` 的 `tapImageControl:` 方法。

### Target-Action的管理
为一个控件对象添加、删除Target-Action的操作我们都已经很熟悉了，主要使用的是以下两个方法：
```objc
// 添加
- (void)addTarget:(id)target action:(SEL)action forControlEvents:(UIControlEvents)controlEvents
 
- (void)removeTarget:(id)target action:(SEL)action forControlEvents:(UIControlEvents)controlEvents
```

如果想获取控件对象所有相关的 `target` 对象，则可以调用 `allTargets` 方法，该方法返回一个集合。集合中可能包含 `NSNull` 对象，表示至少有一个 `nil` 目标对象。

而如果想获取某个 `target` 对象及事件相关的所有 `action`，则可以调用 `actionsForTarget:forControlEvent:` 方法。返回一个可变数组。









































就是这样~~O(∩_∩)O~~