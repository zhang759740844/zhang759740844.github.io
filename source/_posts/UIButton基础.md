title: UIButton 简介
date: 2016/8/11 14:07:12  
categories: IOS
tags: [UIButton]

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
交换了image的位置后，我就想怎么控制image的大小。尝试改变了`UIEdgeInsetsMake`的参数，发现改变`top`和`bottom`可以将图片压缩，但是改变`left``right`图片始终不动。

经过我不断尝试后终于得出了结论：
以横轴为例，只有当**左边距+右边距+图片宽度=button宽度**时，继续增加边距，才会导致图片的压缩。当**左边距+右边距+图片宽度<button宽度**时，增加一边会让图像像另一边移动；同时增加两边，两边抵消，在原处不动。

因此，为什么改变`top`和`bottom`可以将图片压缩呢？因为由于button高度较小，image在纵轴将button填满，此时`top`和`bottom`都为0，但是由于**image高度=button高度**，此时增加`top`和`bottom`就会将图片压缩。当然，如果一个增加，一个等量减小，图片就会像减少那边移动。

而在横轴方面，开始时，**左边距+右边距+图片宽度<button宽度**。图片只会平移直到边距增加到使等式相等才会进行压缩。


就是这样~~O(∩_∩)O~~