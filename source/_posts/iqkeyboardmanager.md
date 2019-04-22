title: IQKeyboardManager 源码解析
date: 2019/4/20 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

IQKeyboardManager 是一个优秀的零行代码解决键盘遮挡的第三方库。现在来学习一下它的实现原理。

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

