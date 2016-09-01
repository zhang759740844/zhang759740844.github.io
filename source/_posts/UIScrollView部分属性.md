title: UIScrollView部分属性
date: 2016/8/4 14:07:12  
categories: IOS
tags: 
	- UI
	- 基本控件
	
---

碰到contentSize、contentOffset不知道用法是什么，就到网上搜索。网上的介绍也比较多，基本大同小异。再此略作摘录、整理。

<!--more-->

## **contentSize**、**contentInset**和**contentOffset**
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