title: UIScrollView详细介绍
date: 2016/9/18 14:07:12  
categories: iOS
tags: 
	- 基本控件
---

本次将参考[iOS 性能优化之 UIScrollView 实践经验](https://my.oschina.net/zhxx/blog/620969#OSC_h3_11)对UIScrollView的使用方式进行详细介绍
<!--more-->

## ScrollView和Auto Layout
`UIScrollView` 在 `Auto Layout` 是一个很特殊的 `view`，对于 `UIScrollView` 的 `subview` 来说，它的 `leading/trailing/top/bottom space` 是**相对于 `UIScrollView`的 `contentSize` 而不是 `bounds` (自身在屏幕上显示的边界，不明白的可以查一下 bounds 和 frame 的区别)来确定的。**

所以，一般的做法是在`UIScrollVIew`和它的`subViews`之间增加一个`content view`。这样可以方便地给 `subview` 提供 `leading/trailing/top/bottom`，方便 `subview` 的布局，并且可以 通过调整`content view` 的 `size`（调整`constraint` 的 `IBOutlet`）来调整 `contentSize`。

### 添加ContentView的注意点
`scrollview`中添加的`contentview`和添加一般的视图不同，一般的视图只要提供`leading/trailing/top/bottom space`就能唯一确定长宽，位置。
而`ScrollView`的`contentView`除了上述四个约束，还需要设置长宽，作为滚动的范围的一部分。

所以整个`scrollview`的`contentSize`的等式应该是：
```objc
scrollView.contentSize.width = (scrollview与contentview之间leading的constant)+（contentView的width）+（scrollview与contentview之间trailing的constant）
```

**另外一点需要强调两点：**
1. `ScrollView`的`bounds`早就确定了下来，在设置`contentview`的宽度的时候，如果设置`contentview.width = superview.width + constant`这里的`superview.width`是指`ScrollView.bounds`而不是`contentSize`
2. 在设置`contentview`的上下左右约束的时候，如果设置`contentview.trailing = superview.trailing + constant`,这里的`superview.trailing`指的是`contentSize.trailing`而不是`bounds`。此时就可以通过上面的公式计算`contentSize`的宽度了。

> Demo参见AutoLayout

## UIScrollView部分属性
### **contentSize**、**contentInset**和**contentOffset**
- contentSize: 就是scrollview可以滚动的区域.
  比如frame = (0 ,0 ,320 ,480) contentSize = (320 ,960)，代表你的scrollview可以上下滚动，滚动区域为frame大小的两倍。
- contentOffset:就是scrollview当前显示区域顶点相对于frame顶点的偏移量。
  比如上个例子你拉到最下面，contentoffset就是(0 ,480)，也就是y偏移了480 
- contentInset:就是scrollview的contentview的顶点相对于scrollview的位置。
  例如你的contentInset = (0 ,100)，那么你的contentview就是从scrollview的(0 ,100)开始显示 

/* 上拉刷新一般实现代码如下 */
```objc
- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate{     
    [_refreshHeaderView egoRefreshScrollViewDidEndDragging:scrollView];  
    float offset=scrollView.contentOffset.y;  
    float contentHeight=scrollView.contentSize.height;  
    float sub=contentHeight-offset;  
    if ((scrollView.height-sub)>20) {//如果上拉距离超过20p，则加载更多数据  
        //[self loadMoreData];//此处在view底部加载更多数据  
    }  
}
```


## UIScrollViewDelegate
`UIScrollViewDelegate` 是 `UIScrollView` 的 `delegate protocol`，`UIScrollView` 有意思的功能都是通过它的 delegate 方法实现的。

### - (void)scrollViewDidScroll:(UIScrollView *)scrollView
这个方法在任何方式触发 `contentOffset` 变化的时候都会被调用（包括用户拖动，减速过程，直接通过代码设置等），可以用于监控 `contentOffset` 的变化，并根据当前的 `contentOffset` 对其他 view 做出随动调整。

### - (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView
用户开始拖动 scroll view 的时候被调用。

### - (void)scrollViewWillEndDragging:(UIScrollView \*)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint \*)targetContentOffset
在 didEndDragging 前被调用，当 willEndDragging 方法中 `velocity` 为 `CGPointZero`（结束拖动时两个方向都没有速度）时，didEndDragging 中的 `decelerate` 为 `NO`，即没有减速过程，willBeginDecelerating 和 didEndDecelerating 也就不会被调用。反之，当 `velocity` 不为 `CGPointZero` 时，scroll view 会以 `velocity` 为初速度，减速直到 `targetContentOffset`。值得注意的是，这里的 `targetContentOffset` 是个指针，可以改变减速运动的目的地，这在一些效果的实现时十分有用。

### - (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate
在用户结束拖动后被调用，`decelerate` 为 `YES` 时，结束拖动后会有减速过程。注，在 didEndDragging 之后，如果有减速过程，scroll view 的 dragging 并不会立即置为 `NO`，而是要等到减速结束之后，所以**这个 dragging 属性的实际语义更接近 scrolling**。

### - (void)scrollViewWillBeginDecelerating:(UIScrollView *)scrollView
减速动画开始前被调用。

### - (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView
减速动画结束时被调用。

这里有一种特殊情况：当一次减速动画尚未结束的时候再次 drag scroll view，didEndDecelerating 不会被调用，**并且这时 scroll view 的 `dragging` 和 `decelerating` 属性都是 `YES`**(因为，此时ScrollView还在滚动，并且还在减速)。新的 dragging 如果有加速度，那么 willBeginDecelerating 会再一次被调用，然后才是 didEndDecelerating；如果没有加速度，虽然 willBeginDecelerating 不会被调用，但前一次留下的 didEndDecelerating 会被调用。

> Demo参见Delegate



## 实例
### Table View 中图片加载逻辑的优化
**优化目的**：滑动时不加载图片，在滑动停止时加载图片

**优化难点**：前面提到，刚开始拖动的时候，`dragging` 为 `YES`，`decelerating` 为 `NO`；`decelerate` 过程中，`dragging` 和 `decelerating` 都为 `YES`；decelerate 未结束时开始下一次拖动，`dragging` 和 `decelerating` 依然都为 `YES`。所以无法简单通过 table view 的 `dragging` 和 `decelerating` 判断是在用户拖动还是减速过程。

所以不能仅通过`decelerating`来判断是否加载图片，还需要自己记录用户拖动状态，在拖动时也要加载图片。（涉及到需要注意的点很多，还是直接看[原文](https://my.oschina.net/zhxx/blog/620969#OSC_h3_11)比较好）

**优化方法**：
1. 每次开始拖动时scrollViewWillBeginDragging，通过`NSArray *cells = [self.tableView visibleCells];`获取屏幕上所有显示的`Cell`，全部加载一遍图片（解决问题三）
2. 利用`scrollViewWillEndDragging: withVelocity: targetContentOffset: `方法，将`targetContentOffset`即最后减速结束后在屏幕上显示的位置，转换为一个`CGRect`，在`CGRect`范围里的`Cell`才在`CellForRowAtIndex`中加载。

> Demo详见LazyLoad

### 分页的几种实现方法
分页类似于 android 中的 viewpager ，利用 UIScrollView 有多种方法实现分页，但是各自的效果和用途不尽相同。

#### pagingEnabled
系统提供的分页方式，实现简单，只需要`_scrollView.pagingEnabled = YES`即可，但是有局限性：
- 只能以 ScrollView 的 frame size 为单位翻页，减速动画阻尼大，减速过程不超过一页。
- 需要一些 hacking 实现 bleeding 和 padding（即页与页之间有 padding，在当前页可以看到前后页的部分内容）

Sample 中 Pagination 有简单实现 bleeding 和 padding 效果的代码，主要的思路是：
- 让 scroll view 的宽度为 page 宽度 + padding，并且设置 `clipsToBounds` 为 `NO`,这样前后不在 scrollView 中的部分就能显示一部分出来。见下图:
- 这样虽然能看到前后页的内容，但是无法响应 touch，所以需要另一个覆盖期望的可触摸区域的 view 来实现类似 touch bridging 的功能

![UIScrollView_pagingEnabled](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/UIScrollView_pagingEnabled.png?raw=true)

适用场景：上述局限性同时也是这种实现方式的优点，比如一般 App 的引导页（教程），Calendar 里的月视图，都可以用这种方法实现。

#### Snap
核心算法是在 `didEndDragging` 方法中，通过当前 contentOffset 计算最近的整数页及其对应的 contentOffset，通过动画 snap 到该页。这个方法实现的效果都有个通病，就是最后的 snap 会在 decelerate 结束以后才发生，总感觉很突兀。使用体验不如下面的方法。

#### 修改targetContentOffset
通过修改 `scrollViewWillEndDragging: withVelocity: targetContentOffset:` 方法中的 `targetContentOffset` 直接修改目标 `offset` 为整数页位置。其中核心代码：
```objc
- (CGPoint)nearestTargetOffsetForOffset:(CGPoint)offset
{    CGFloat pageSize = BUBBLE_DIAMETER + BUBBLE_PADDING;    NSInteger page = roundf(offset.x / pageSize);    CGFloat targetX = pageSize * page;    return CGPointMake(targetX, offset.y);
}

- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset
{    CGPoint targetOffset = [self nearestTargetOffsetForOffset:*targetContentOffset];
    targetContentOffset->x = targetOffset.x;
    targetContentOffset->y = targetOffset.y;
}
```

适用场景：方法 2 和 方法 3 的原理近似，效果也相近，适用场景也基本相同，但方法 3 的体验会好很多，snap 到整数页的过程很自然，或者说用户完全感知不到 snap 过程的存在。这两种方法的减速过程流畅，适用于一屏有多页，但需要按整数页滑动的场景；也适用于如图表中自动 snap 到整数天的场景；还适用于每页大小不同的情况下 snap 到整数页的场景（不做举例，自行发挥，其实只需要修改计算目标 offset 的方法）。

> Demo详见Pagination

#### UIPageControl

首先声明这个和 ScrollView 如何设置分页没有关系。这个组件是 iOS 在分页中经常用的表示页数的小白点。简单记录一下怎么使用。

使用起来其实蛮简单的，就当一个 View 来使用就行了：

```objc
    _pageControl = [[UIPageControl alloc]initWithFrame:CGRectMake(SCREEN_WIDTH/2-75, SCREEN_HEIGHT-30-20, 150, 20)];
    _pageControl.numberOfPages = 3;
    _pageControl.currentPage = 0;
    _pageControl.pageIndicatorTintColor = [UIColor colorWithHexString:@"E1E1E1"];
    _pageControl.currentPageIndicatorTintColor = COLOR_MAINRED;
    [self.view addSubview:_pageControl];
```

几个属性也比较易懂，基本上就设置好了，当然这里还没有设置和 Scrollview 的联动：

```objc
//监听scrollView滚动事件
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    int page = scrollView.contentOffset.x / SCREEN_WIDTH;
    _pageControl.currentPage = page;
}
```

监听 Scrollview 的滚动事件就可以了。

### 重用
大部分的 iOS 开发应该都清楚 `UITableView` 的 cell 重用机制，这种重用机制减少了内存开销也提高了 performance，`UIScrollView` 作为 `UITableView` 的父类，在很多场景中也很适合应用重用机制。

可以参照 `UITableView` 的 cell 重用机制，总结重用机制如下：
- 维护一个重用队列
- 当元素离开可见范围时，`removeFromSuperview` 并加入重用队列（enqueue）
- 当需要加入新的元素时，先尝试从重用队列获取可重用元素（dequeue）并且从重用队列移除
- 如果队列为空，新建元素
- 将新建元素的view通过`addSubView`添加至 `contentView`上
- 这些一般都在 `scrollViewDidScroll:` 方法中完成

Demo中演示了一个在scrollView中添加多个重用ViewController的场景，具体重用方法参见Demo

---
这里要说明一下Demo中用到的`addChildViewController`。

苹果在iOS 5 添加了一系列的方法，希望我们在使用`addSubView`时，同时调用`[self addChildViewController:child]`方法将sub view对应的viewController也加到当前ViewController的管理中。

对于那些当前暂时不需要显示的subview，只通过`addChildViewController`把subViewController加进去；需要显示时再调用`transitionFromViewController`方法。将其添加进入底层的ViewController中。

这样做的好处：
1. 对页面中的逻辑更加分明了。相应的View对应相应的ViewController。
2. 当某个子View没有显示时，将不会被Load，减少了内存的使用。
3. 当内存紧张时，没有Load的View将被首先释放，优化了程序的内存释放机制。
4. 可以调用`transitionFromViewController`等系统方法，ios对于显示以及动画做了比较好的封装
5. 相当于对于 ViewController 实例的保存。实例都保存在`self.childViewControllers`中，需要使用的时候从中取出即可。

当然，像本例中在 ScrollView 中添加 ViewController ，只需要对 ViewController 进行复用，但是用不到其提供的系统方法时，其实只要使用一个数组将各个 ViewController 保存起来就可以了。

其实这些都不重要，如果不需要对 ViewController 做复用或者一些其他操作，我们连ViewController的实例都不必保存。我们只需要在需要显示的时候`[self.contentView addSubview:vc.view];`,不需要显示的时候`[vc.view removeFromSuperview];`，就可以达到显示和相应事件的需求了。

---

> Demo详见Reuse

### 联动
所谓联动，就是当 A 滚动的时候，在 **`scrollViewDidScroll:`** 里根据 A 的 `contentOffset` 动态计算 B 的 `contentOffset` 并设给 B。同样对于非 scroll view 的 C，也可以动态计算 C 的 frame 或是 transform实现视差滚动或者其他高级动画，这在现在许多应用的引导页面里会被用到。

Demo中的核心代码:
```objc
- (void)scrollViewDidScroll:(UIScrollView *)scrollView{
    if (scrollView == self.titleScrollView) {
        CGFloat contentX = self.titleScrollView.contentOffset.x / self.titleScrollView.frame.size.width * self.contentScrollView.frame.size.width;
        self.contentScrollView.contentOffset = CGPointMake(contentX, 0.0);
        CGFloat transX = self.titleScrollView.contentOffset.x / (self.titleScrollView.contentSize.width - self.titleScrollView.frame.size.width) * (self.backgroundImage.frame.size.width - self.view.frame.size.width);
        transX = MAX(0.0, transX);
        transX = MIN(self.backgroundImage.frame.size.width - self.view.frame.size.width, transX);
        self.backgroundImage.transform = CGAffineTransformMakeTranslation(-transX, 0.0);
    }
}
```
通过`titleScrollView`计算`contentX`和`transX`来分别控制`contentScrollView`以及`backgroundImage`这两个View的位置变化。

> Demo参见Parallax

## 总结
ScrollView看似简单，其实还是有非常多的地方值得去挖掘的。使用好，能够实现很多非常酷炫的效果。上面的原理和应用涉及到的点很多，还需要细细琢磨，学以致用.

> 上述所有Demo均在UIScrollViewDemo文件夹中






