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
#import "IQTextView.h"

IQTextView *textView = [[IQTextView alloc] init];
textView.placeholder = @"这是一个placeholder";
textView.placeholderTextColor = [UIColor redColor];
```

### IQKeyboardReturnKeyHandler

这个文件可以帮助我们将键盘上的 return 键变为 next 键，点击进入下一个输入框。当到最后一个输入框的时候，变为 Done，点击收起键盘。

```objc
#import "IQKeyboardReturnKeyHandler.h"

@property (nonatomic, strong) IQKeyboardReturnKeyHandler *returnKeyHandler;

- (void)viewDidLoad:(BOOL)animated{
  [super viewDidLoad:animated];
  self.returnKeyHander = [[IQKeyboardReturnKeyHandler alloc] initWithViewController:self];
  self.returnKeyHander.delegate = self;
}
```

使用非常简单。只要传入当前 textfield 所在的控制器即可。可以通过设置 `IQKeyboardReturnHandler` 的 delegate 设置所有 textfield 的 delegate。也可以自己设置每个 textfield 的 delegate。

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

### IQKeyboardReturnKeyHandler

IQKeyboardReturnKeyHandler 主要用来解决多个 textfield 的时候点击 return 跳到下一个 textfield 的问题。没有看过源码的时候觉得是一个非常神奇的功能。其实实现方式简单点说就是获取当前 UIViewController 内的所有 textfield，然后对他们按照位置排序。点击 return 就让下一个 textfield 获取焦点。

#### 初步处理

初步处理的过程中，会把 UIViewController 中的所有 textfield 全都拿出来，保存基本信息。目的是当 `IQKeyboardReturnKeyHandler` 实例销毁的时候，将所有 textfield 恢复如初。

首先看入口方法：

```objc
// 把 controller 中的所有 textfield 添加到这个类中，所有代理由这个类托管。
-(instancetype)initWithViewController:(nullable UIViewController*)controller {
    self = [super init];

    if (self)
    {
        textFieldInfoCache = [[NSMutableSet alloc] init];

        if (controller.view)
        {
            [self addResponderFromView:controller.view];
        }
    }

    return self;
}

// 获取当前视图的所有子视图中的 textfield 并添加到数组中
-(void)addResponderFromView:(UIView*)view {
    NSArray<UIView*> *textFields = [view deepResponderViews];

    for (UIView *textField in textFields)  [self addTextFieldView:textField];
}
```

可以看到一个熟悉的方法 `deepResponderViews`，也就是说，初始化的时候，从 UIViewController 中找到了所有的 textfield，并且通过 `addTextFieldView` 方法把他们保存起来。`addTextFieldView` 方法把 textfield 的 `originalReturnKeyType` `delegate` 保存了起来，转为了一个 modal，并把它们的 delegate 设置为了自己。这样 textfield 的所有事件都会由 `IQKeyboardReturnKeyHandler` 实例接管。

```objc
// 把 TextField 转为 Modal 添加到 cache 中
-(void)addTextFieldView:(UIView*)view {
    IQTextFieldViewInfoModal *modal = [[IQTextFieldViewInfoModal alloc] initWithTextFieldView:view textFieldDelegate:nil textViewDelegate:nil originalReturnKey:UIReturnKeyDefault];

    if ([view isKindOfClass:[UITextField class]])
    {
        UITextField *textField = (UITextField*)view;
        modal.originalReturnKeyType = textField.returnKeyType;
        modal.textFieldDelegate = textField.delegate;
        [textField setDelegate:self];
    }
    else if ([view isKindOfClass:[UITextView class]])
    {
        UITextView *textView = (UITextView*)view;
        modal.originalReturnKeyType = textView.returnKeyType;
        modal.textViewDelegate = textView.delegate;
        [textView setDelegate:self];
    }

    [textFieldInfoCache addObject:modal];
}
```

#### 接管 textfield 的 textfieldDidBeginEditing 方法

IQKeyboardReturnKeyHandler 实例接管了 textfield 的相关方法。在 `textfieldDidBeginEditing` 中。它会将除最后一个 textfield 之外的所有 textfield 的 returnkeytype 设置为 next，最后一个设置为 return。

具体看代码：

```objc
- (void)textFieldDidBeginEditing:(UITextField *)textField
{
    [self updateReturnKeyTypeOnTextField:textField];
		// 省略了调用代理方法的代码	
  ...
}
```

```objc
-(void)updateReturnKeyTypeOnTextField:(UIView*)textField {
  	// 省略了关于 UITableView 相关的搜索逻辑
  	...
    textFields = [textField responderSiblings];
    switch ([[IQKeyboardManager sharedManager] toolbarManageBehaviour])
    {
        case IQAutoToolbarByTag:
            textFields = [textFields sortedArrayByTag];
            break;
        case IQAutoToolbarByPosition:
            textFields = [textFields sortedArrayByPosition];
            break;
        default:
            break;

    [(UITextField*)textField setReturnKeyType:(([textFields lastObject] == textField)    ?   self.lastTextFieldReturnKeyType :   UIReturnKeyNext)];
}
```

`updateReturnKeyTypeOnTextField` 方法中将被编辑的 textfield 的兄弟 textfield 全都拿到，然后按照所处位置排序，得到一个排序好的 `textFields` 兄弟数组，把他们最后一个设置为 Return 类型，其他的都设置为 Next 类型。

其中省略了一部分关于 UITableView 的处理逻辑。设想一下，如果一个 textfield 处于 UITableView 的一个 cell 中，那么应该这个 UITableView 的其他 cell 也会有 textfield。这些 textfield 虽然不是在一个视图中，但应该也是同级的兄弟 textfield。因此，IQKeyboardManager 针对这种情况会对 UITableView 进行深搜，拿到所有的 textfield。

这里省略 UITableView 相关逻辑是因为我们只需要知道设置 textfield 的 returnkeytype 的关键点在于在开始编辑的时候找到所有 textfield 的兄弟 textfield 即可。UITableView 相关逻辑只是对这个目的的补充。

#### 接管 textfield 的 textFieldShouldReturn 方法

和 `textfieldDidBeginEditing` 相对应的，当点击 next 键的时候，会将焦点置于下一个 textfield：

```objc
-(BOOL)textFieldShouldReturn:(UITextField *)textField
{
    id<UITextFieldDelegate> delegate = self.delegate;
    
    if ([delegate respondsToSelector:@selector(textFieldShouldReturn:)])
    {
        BOOL shouldReturn = [delegate textFieldShouldReturn:textField];

        if (shouldReturn)
        {
            shouldReturn = [self goToNextResponderOrResign:textField];
        }
        
        return shouldReturn;
    }
    else
    {
        return [self goToNextResponderOrResign:textField];
    }
}
```

```objc
-(BOOL)goToNextResponderOrResign:(UIView*)textField {
  	// 省略 UITableView 相关逻辑
  	...
      
    textFields = [textField responderSiblings];
    
    switch ([[IQKeyboardManager sharedManager] toolbarManageBehaviour])
    {
        case IQAutoToolbarByTag:
            textFields = [textFields sortedArrayByTag];
            break;
        case IQAutoToolbarByPosition:
            textFields = [textFields sortedArrayByPosition];
            break;
        default:
            break;
    }
        
    NSUInteger index = [textFields indexOfObject:textField];
    
    if (index != NSNotFound && index < textFields.count-1) {
        [textFields[index+1] becomeFirstResponder];
        return NO;
    } else {
        [textField resignFirstResponder];
        return YES;
    }
}
```

其实理解了设置 returnkeytype 的逻辑，这里设置下一个响应者的逻辑也就明了了。还是获得排序后的 textfield 数组，只要把下一个 textfield 设置为第一响应者就可以了。



## 小技巧

到最后了总结一下看 IQKeyboardManager 源码学到的一些技巧。

1. 对一个数组排序：

```objc
[SomeViewArray sortedArrayUsingComparator:^NSComparisonResult(UIView *view1, UIView *view2) {
        
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
```

2. 通过响应链获得当前视图的 ViewController

```objc
-(UIViewController*)viewContainingController
{
    UIResponder *nextResponder =  self;
    
    do
    {
        nextResponder = [nextResponder nextResponder];

        if ([nextResponder isKindOfClass:[UIViewController class]])
            return (UIViewController*)nextResponder;

    } while (nextResponder);

    return nil;
}
```

