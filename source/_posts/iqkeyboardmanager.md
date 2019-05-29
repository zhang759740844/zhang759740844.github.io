title: IQKeyboardManager 源码解析
date: 2019/4/20 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

IQKeyboardManager 是一个优秀的零行代码解决键盘遮挡的第三方库。在没有看过源码的时候是我认为的最有魔力的第三方库。现在我们就要揭开它的面纱。

<!--more-->

## 使用方式

### IQTextView

`IQTextView` 是一个提供了 placeholder 的 `UITextView`。可以设置它文字和颜色：

```objc
IQTextView *textView = [[IQTextView alloc] init];
textView.placeholder = @"这是一个placeholder";
textView.placeholderTextColor = [UIColor redColor];
```



## 源码解析

### IQTextView

`IQTextView` 主要就是在 `UITextView` 的基础上添加了一个 `UILabel`。实现起来也非常简单。

placeholder 主要关注两件事，一是 placeholder 的位置，二是 placeholder 何时隐藏。

首先看 placeholder 的位置，它通过 `sizeThatFits` 方法获取到占位符的大小：

```objc
// 布局方法
-(void)layoutSubviews
{
    [super layoutSubviews];
    self.placeholderLabel.frame = [self placeholderExpectedFrame];
}

// placeholder 的 inset
-(UIEdgeInsets)placeholderInsets
{
    return UIEdgeInsetsMake(self.textContainerInset.top, self.textContainerInset.left + self.textContainer.lineFragmentPadding, self.textContainerInset.bottom, self.textContainerInset.right + self.textContainer.lineFragmentPadding);
}

-(CGRect)placeholderExpectedFrame
{
    UIEdgeInsets placeholderInsets = [self placeholderInsets];
    CGFloat maxWidth = CGRectGetWidth(self.frame)-placeholderInsets.left-placeholderInsets.right;
    
    CGSize expectedSize = [self.placeholderLabel sizeThatFits:CGSizeMake(maxWidth, CGRectGetHeight(self.frame)-placeholderInsets.top-placeholderInsets.bottom)];
    
    return CGRectMake(placeholderInsets.left, placeholderInsets.top, maxWidth, expectedSize.height);
}
```

再看 placeholder 何时隐藏。何时隐藏？只要当前 textView 的 text 不为空就要隐藏。那么怎么知道不为空呢？注册监听 `UITextViewTextDidChangeNotification` 的通知

```objc
-(void)initialize
{
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(refreshPlaceholder) name:UITextViewTextDidChangeNotification object:self];
}

-(void)refreshPlaceholder
{
    /// 如果有 text 或者 attributedText 那么就显示否则隐藏
    if([[self text] length] || [[self attributedText] length])
    {
        [_placeholderLabel setAlpha:0];
    }
    else
    {
        [_placeholderLabel setAlpha:1];
    }
    
    [self setNeedsLayout];
    [self layoutIfNeeded];
}
```

### 分类文件

#### 数组分类

数组分类中包含该两个方法，通过 `tag` 大小或者通过位置对 UIView 排序：

```objc
- (NSArray<UIView*>*)sortedArrayByTag
- (NSArray<UIView*>*)sortedArrayByPosition
```

这个方法中我们可以学习一个排序的方法：

```objc
- (NSArray<UIView*>*)sortedArrayByPosition
{
    return [self sortedArrayUsingComparator:^NSComparisonResult(UIView *view1, UIView *view2) {
        
        CGFloat x1 = CGRectGetMinX(view1.frame);
        CGFloat y1 = CGRectGetMinY(view1.frame);
        CGFloat x2 = CGRectGetMinX(view2.frame);
        CGFloat y2 = CGRectGetMinY(view2.frame);
        
        if (y1 < y2)  return NSOrderedAscending;
        
        else if (y1 > y2) return NSOrderedDescending;
        
        //Else both y are same so checking for x positions
        else if (x1 < x2)  return NSOrderedAscending;
        
        else if (x1 > x2) return NSOrderedDescending;
        
        else    return NSOrderedSame;
    }];
}
```

通过 `sortedArrayUsingComparator` 传入一个比较大小的 block。先根据 y 轴比较大小，y 轴相等的时候再比较 x 轴大小。这样得到的就是一个根据该方法排序的数组。

#### UIScrollView 分类

UIScrollView 分类中添加了两个分类属性：

```objc
// 标识该 UIScrollView 是否可以调整 contentOffset 来达到调整 textfield 位置的目的。
// 默认是 NO，表示该 UIScrollView 可以被调整。
// 如果设置为 YES，表示当前视图不能移动，就会找上级视图移动
@property(nonatomic, assign) BOOL shouldIgnoreScrollingAdjustment;

// 是否保存调整位置前的 UIScrollView 的 contentOffset 的值。如果保存，那么恢复后 UIScrollView 将滚回原来的位置。默认是 NO，不保存
@property(nonatomic, assign) BOOL shouldRestoreScrollViewContentOffset;
```

#### UITextField 分类

其实是添加在 UIView 上的几个属性：

```objc
// 设置键盘弹出后，textfield 到键盘的距离
@property(nonatomic, assign) CGFloat keyboardDistanceFromTextField;
// 如果设置为 YES，那么上一个 textfield 点击 next 之后，就不会跳到这个 textfield 上
@property(nonatomic, assign) BOOL ignoreSwitchingByNextPrevious;
// 焦点在这个 textfield 上时，是否可以点击外部取消焦点
@property(nonatomic, assign) IQEnableMode shouldResignOnTouchOutsideMode;
```

#### UIView 分类

UIView 的分类下的方法分为两类，一类是获取 UIView 所在的 UIViewController，一类是获取当前视图下的 UITextField：

```objc
// 获得 UIView 的控制器,通过响应链获取
@property (nullable, nonatomic, readonly, strong) UIViewController *viewContainingController;
// 获得最上层的视图控制器
@property (nullable, nonatomic, readonly, strong) UIViewController *topMostController;
// 获取当前视图的所有兄弟 textfield
@property (nonnull, nonatomic, readonly, copy) NSArray<__kindof UIView*> *responderSiblings;
// 把当前视图下的所有的 textfield 都收集起来，包含视图的子视图
@property (nonnull, nonatomic, readonly, copy) NSArray<__kindof UIView*> *deepResponderViews;
```

### IQKeyboardManager