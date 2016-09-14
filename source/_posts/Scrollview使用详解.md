title: UIScrollView详细介绍
date: 2016/8/4 14:07:12  
categories: IOS
tags: 
	- UI
	- 基本控件
	
---

UIScrollView是iOS开发中最常用的UI组件，本次将参考[iOS 性能优化之 UIScrollView 实践经验](https://my.oschina.net/zhxx/blog/620969#OSC_h3_11)对UIScrollView的使用方式进行详细介绍
<!--more-->

## ScrollView和Auto Layout
`UIScrollView` 在 `Auto Layout` 是一个很特殊的 `view`，对于 `UIScrollView` 的 `subview` 来说，它的 `leading/trailing/top/bottom space` 是**相对于 `UIScrollView`的 `contentSize` 而不是 `bounds` 来确定的。**

所以，一般的做法是在`UIScrollVIew`和它的`subViews`之间增加一个`content view`。这样可以方便地给 `subview` 提供 `leading/trailing/top/bottom`，方便 `subview` 的布局，并且可以 通过调整`content view` 的 `size`（调整`constraint` 的 `IBOutlet`）来调整 `contentSize`。

### 添加ContentView的注意点
`scrollview`中添加的`contentview`和添加一般的视图不同，一般的视图只要只要提供`leading/trailing/top/bottom space`就能唯一确定长宽，位置。
而`ScrollView`的`contentView`除了上述四个约束，还需要设置长宽，作为滚动的范围的一部分。

所以整个`scrollview`的`contentSize`的等式应该是：
```
scrollView.contentSize.width = (scrollview与contentview之间leading的constant)+（contentView的width）+（scrollview与contentview之间trailing的constant）
```

**另外一点需要强调两点：**
1. `ScrollView`的`bounds`早就确定了下来，在设置`contentview`的宽度的时候，如果设置`contentview.width = superview.width + constant`这里的`superview.width`是指`ScrollView.bounds`而不是`contentSize`
2. 在设置`contentview`的上下左右约束的时候，如果设置`contentview.trailing = superview.trailing + constant`,这里的`superview.trailing`指的是`contentSize.trailing`而不是`bounds`。此时就可以通过上面的公式计算`contentSize`的宽度了。

