title: UILabel高度控制
date: 2016/9/12 14:07:12  
categories: iOS 
tags: 
​	- UI
​	

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

## boundingRectWithSize

`boundingRectWithSize` 其实和上面的方法差不多，但是前面的方法需要拿到 label 的实例。下面这个方法则是通过 String 直接计算的：

```objc
NSString *text = @"Today is a fine day";
UIFont *font = [UIFont systemFontOfSize:30];
CGRect suggestedRect = [text boundingRectWithSize:CGSizeMake(800, MAXFLOAT) options:NSStringDrawingUsesFontLeading attributes:@{ NSFontAttributeName : font } context:nil];
NSLog(@"size = %@", NSStringFromCGSize(suggestedRect.size));
```

