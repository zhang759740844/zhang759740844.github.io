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

创建一个自定义类型的 button

```objc
UIButton *btn1 = [[UIButton buttonWithType:UIButtonTypeCustom];
CGRect btn1Frame = CGRectMake(50, 50, 200, 100);
btn1.frame = btn1Frame;
[self.view addSubview:btn1];
```



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
- UIControlStateNormal 常态
- UIControlStateHighlighted 高亮
- UIControlStateDisabled 禁用
- UIControlStateSelected 选中

其中需要说明的是，高亮就是点击时的状态。其实还有一种`UIControlStateSelected | UIControlStateHighlighted`这个组合是选中时候的高亮状态，也是比较有用的。

### 设置button选中图片
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

好的，这样就完成了切换状态的过程。（上面说的是两种高亮状态下的设置。完整的代码应该是这样的：

```objc
// 未选中状态
[button setImage:[UIImage imageNamed:@"like"] forState:UIControlStateNormal];
// 选中状态
[button setImage:[UIImage imageNamed:@"like"] forState: UIControlStateHighlighted];
// 从未选中到选中的高亮状态
[button setImage:[UIImage imageNamed:@"like_selected"] forState:UIControlStateSelected];
// 从选中到未选中的高亮状态
[button setImage:[UIImage imageNamed:@"like_selected"] forState:UIControlStateSelected | UIControlStateHighlighted];
```

### 取消图片的点击高亮

图片的点击会有一个高亮状态，如果你不想要有这个状态，那么需要取消点击高亮，可以：

```objc
btn.adjustsImageWhenHighlighted = NO;
```

### 设置title 和 image 的位置

我们可以通过 `UIEdgeInserts` 的方式修改位置，但是这种方式经常会压缩拉长 image，所以更好的方式是通过继承 UIButton，并且重写 `layoutSubViews` 方法的方式：

```swift
override func layoutSubviews() {
    super.layoutSubviews()
    imageView?.bounds = CGRect(x: 0, y: 0, width: 50, height: 50)
    titleLabel?.bounds = CGRect(x: 0, y: 0, width: 50, height: 50)
}
```

























就是这样~~O(∩_∩)O~~