title: UILabel高度控制
date: 2016/9/12 14:07:12  
categories: iOS 
tags: 
	- UI
	

---

在[tableview自适应高度](https://zhang759740844.github.io/2016/08/26/UITableview自适应高度/)一篇中学习了如何用`systemLayoutSizeFittingSize:`方法得到UITableView的Cell的高度。本文将探究一般情况下如何得到UILabel的高度。

<!--more-->

## sizeThatFits的使用
`sizeThatFits`是UIView中的方法：
```objc
- (CGSize)sizeThatFits:(CGSize)size;
```

官方文档的注释：
>return 'best' size to fit given size. does not actually resize view. Default is return existing view size
也就是说，该方法将会根据传入的`size`,计算得到最佳的UIView的宽高，直接返回，不对View做任何修改。

UILabel会根据传入的`size`的`width`自动换行，得到`height`后，将这两个值作为估算出的`CGSize`返回。
如果传入的是`CGSizeZero`相当于不设置换行宽度，一行到底。

由于`SizeThatFits`只估算，不修改。在得到估算值后，需要手动设置UILabel的`frame`或者`constraint`：
```objc
- (void) useSizeThatFitsZeroWithLabel:(UILabel *)label{
    //使用CGSizeZero相当于不设置换行宽度，一行到底
    CGSize size = [label sizeThatFits:CGSizeZero];
    label.frame = CGRectMake(label.frame.origin.x, label.frame.origin.y, size.width, size.height);
}

- (void) useSizeThatFitsCustomWithLabel:(UILabel *)label{
    //使用自定义的CGSize，会根据size的宽度进行换行
    CGSize size = [label sizeThatFits:CGSizeMake(50, 50)];
    label.frame = CGRectMake(label.frame.origin.x, label.frame.origin.y, size.width, size.height);
}
```

## SizeToFit
`SizeToFit`的方法：
```objc
- (void)sizeToFit;  
```

`SizeToFit`的文档注释：
> calls sizeThatFits: with current view bounds and changes bounds size.
也就是说，它会调用`SizeThatFits`,并且直接设置View的`bounds`。

既然要调用`sizeThatFits`那就需要传入一个`CGSize`。这个`CGSize`就是设置View的`frame`时传入的`height`,`width`。





## frame与constraint
在代码添加View时，init都会使用`initWithFrame:`方法。在AutoLayout后，`constraint`被使用，是为了适配不同屏幕的机器。

- 设置好View之间的`constraint`后，一个View的约束改变，将会联动改变其他View的位置、大小。
- 设置好View的`frame`或者`bound`后，改变View的`frame`，不会对其它View造成任何影响。
- 在设置好`constraint`后，显示的View就以约束为准。修改`frame`，View的显示没有任何改变，虽然`frame`确实变了。

```objc
NSLog(@"改变前%f,%f,%f,%f",_label.frame.size.height,_label.frame.size.width,_label.frame.origin.x,_label.frame.origin.y);
_label.frame = CGRectMake(0, 0, 200, 200);
NSLog(@"改变后%f,%f,%f,%f",_label.frame.size.height,_label.frame.size.width,_label.frame.origin.x,_label.frame.origin.y);

2016-09-12 15:32:44.458 Parallax[2126:578089] 66.000000,147.000000,180.000000,117.000000
2016-09-12 15:32:44.459 Parallax[2126:578089] 改变后200.000000,200.000000,0.000000,0.000000
```


## 与systemLayoutSizeFittingSize的比较
`systemLayoutSizeFittingSize`方法是在整个view的约束已经确定，通过计算UILabel的高度(或许其中也调用了`sizeThatFits`吧)，得到最适合的supview的高度。

`sizetofit`和`sizethatfits`则是在supview的高度确定的情况下，计算得到UILabel的高度，然后通过手动设置frame或者约束条件，将其添加到supview上。

> Demo 详见 UIScrollViewDemo/Parallax