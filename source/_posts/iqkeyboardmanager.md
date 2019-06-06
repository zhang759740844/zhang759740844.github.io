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

### IQKeyboardManager

#### 在某个页面禁用 IQKeyboardManager

```objc
 - (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    //写入这个方法后,这个页面将没有这种效果
    [IQKeyboardManager sharedManager].enable = NO;
}
- (void)viewWillDisappear:(BOOL)animated{
    [super viewWillDisappear:animated];
    //最后还设置回来,不要影响其他页面的效果
    [IQKeyboardManager sharedManager].enable = YES;
}
```

除了上面的直接禁用和启用，IQKeyboardManager 也可以设置在禁用的时候在部分 ViewController 上启用，或者在启动的时候在部分 ViewController 上禁用：

```objc
// 在整体禁用的时候可以启动 IQKeyboardManager 的类
[[IQKeyboardManager sharedManager].enabledDistanceHandlingClasses addObject: yourViewControllerClass];
// 在整体启动的时候需要禁用 IQKeyboardManager 的类
[[IQKeyboardManager sharedManager].disabledDistanceHandlingClasses addObject: yourViewControllerClass];
```



#### 点击空白处可以隐藏键盘

```objc
[IQKeyboardManager sharedManager].shouldResignOnTouchOutside = YES;
```

#### 隐藏键盘上的 toolbar

```objc
[IQKeyboardManager sharedManager].enableAutoToolbar = NO;
```

除了这种一刀切的隐藏或者显示 toolbar 之外，IQKeyboardManager 还提供了两个数组属性，用于标识特例 ViewController：

```objc
@property(nonatomic, strong, nonnull, readwrite) NSMutableSet<Class> *disabledToolbarClasses;
@property(nonatomic, strong, nonnull, readwrite) NSMutableSet<Class> *enabledToolbarClasses;
```

这两个属性分别可以设置在 `enableAutoToobar` 为 YES 的时候，不显示 toobar 的 ViewController；`enableAutoToobar` 为 NO 的时候，显示 toobar 的 ViewController。

`disableToolbarClasses` 默认为：

```objc
[UIAlertController, _UIAlertControllerTextFieldViewController]
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

### IQKeyboardManager

#### 注册

IQKeyboardManager 通过 `+(void)load` 方法自动创建自身：

```objc
+(void)load
{
    //Enabling IQKeyboardManager. Loading asynchronous on main thread
    [self performSelectorOnMainThread:@selector(sharedManager) withObject:nil waitUntilDone:NO];
}
```

我们常说不要在 load 方法中做太多耗时操作，会影响应用的启动速度。所以，我们可以把要做的初始化操作异步去执行。下面来看初始化方法

##### 注册通知

注册的通知主要包含两部分。一部分是键盘的弹出与隐藏，另一部分是 `UITextField` 和 `UITextView` 的编辑的回调。具体通知的处理方法后文解析。

```objc
-(void)registerAllNotifications
{
    //  注册键盘相关通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardWillShow:) name:UIKeyboardWillShowNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardDidShow:) name:UIKeyboardDidShowNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardWillHide:) name:UIKeyboardWillHideNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardDidHide:) name:UIKeyboardDidHideNotification object:nil];

    //  注册 UITextField 的编辑通知
    [self registerTextFieldViewClass:[UITextField class]
     didBeginEditingNotificationName:UITextFieldTextDidBeginEditingNotification
       didEndEditingNotificationName:UITextFieldTextDidEndEditingNotification];

    //  注册 UITextView 的编辑通知
    [self registerTextFieldViewClass:[UITextView class]
     didBeginEditingNotificationName:UITextViewTextDidBeginEditingNotification
       didEndEditingNotificationName:UITextViewTextDidEndEditingNotification];
}

// 注册开始编辑和结束编辑的通知
-(void)registerTextFieldViewClass:(nonnull Class)aClass
  didBeginEditingNotificationName:(nonnull NSString *)didBeginEditingNotificationName
    didEndEditingNotificationName:(nonnull NSString *)didEndEditingNotificationName
{
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(textFieldViewDidBeginEditing:) name:didBeginEditingNotificationName object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(textFieldViewDidEndEditing:) name:didEndEditingNotificationName object:nil];
}
```

> 注册之外的所有逻辑都在这些通知方法内

##### 创建一个 `UITapGestureRecognizer`

这个手势用来在点击 `UITextField` 以外的区域的时候收起键盘

```objc
// 为点击屏幕取消第一响应者这个功能创建一个手势
strongSelf.resignFirstResponderGesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(tapRecognized:)];
// 设置该手势不取消事件传递
strongSelf.resignFirstResponderGesture.cancelsTouchesInView = NO;
// 设置手势代理为自身
[strongSelf.resignFirstResponderGesture setDelegate:self];
// 将手势默认设为不启用
strongSelf.resignFirstResponderGesture.enabled = strongSelf.shouldResignOnTouchOutside;
// 点击外部区域是否取消第一响应者
[self setShouldResignOnTouchOutside:NO];

- (void)tapRecognized:(UITapGestureRecognizer*)gesture  // (Enhancement ID: #14)
{
    if (gesture.state == UIGestureRecognizerStateEnded)
    {
        //Resigning currently responder textField.
        [self resignFirstResponder];
    }
}
```

##### 配置键盘遮挡相关的一些类

下面的这些配置用于设置 Textfield 在哪些类中 IQKeyboardManager 能够启用，或者关闭。一般使用场景不多。见名思意。

```objc
strongSelf.disabledDistanceHandlingClasses = [[NSMutableSet alloc] initWithObjects:[UITableViewController class],[UIAlertController class], nil];
strongSelf.enabledDistanceHandlingClasses = [[NSMutableSet alloc] init];

strongSelf.disabledToolbarClasses = [[NSMutableSet alloc] initWithObjects:[UIAlertController class], nil];
strongSelf.enabledToolbarClasses = [[NSMutableSet alloc] init];

strongSelf.toolbarPreviousNextAllowedClasses = [[NSMutableSet alloc] initWithObjects:[UITableView class],[UICollectionView class],[IQPreviousNextView class], nil];

strongSelf.disabledTouchResignedClasses = [[NSMutableSet alloc] initWithObjects:[UIAlertController class], nil];
strongSelf.enabledTouchResignedClasses = [[NSMutableSet alloc] init];
strongSelf.touchResignedGestureIgnoreClasses = [[NSMutableSet alloc] initWithObjects:[UIControl class],[UINavigationBar class], nil];
```

#### textFieldViewDidBeginEditing

在这个开始编辑的方法中，主要做了两件事：

1. 根据需要添加或移除 toolbar
2. 根据需要调整视图偏移

##### 添加或移除 toolbar

首先会调用 `privateIsEnableAutoToobar` 方法，判断是否需要显示 toobar。这个判断逻辑就是根据外部设置的 `enable` 变量的值，然后还有前面提到的当前 textfield 所在的 ViewController 是否是 `enable` 的特例。伪代码如下：

```objc
- (Bool) privateIsEnableAutoToobar {
  if (开启IQKeyboardManager) {
    if (textfield 在禁止的 ViewController 中) {
      return NO;
    }
} else {
    if (textfield 在允许的 ViewController 中，并且不是 UIAlertController 和 TextFieldViewController ) {
      return YES;
    }
	}
}
```

如果允许添加。那么就会调用 `addToolbarIfRequired` 方法。主要是在 toolbar 中添加各种 button。伪代码如下：

```objc
- (void)addToolbarIfRequired {
  if (textfield 能够添加 inputAccessoryView && textfield 的 inputAccessoryView 为 nil) {
    if (有自定义的图片 toolbarDoneBarButtonItemImage) {
      用这个 toolbarDoneBarButtonItemImage 初始化一个 toolbar 右边的 done 按钮
    } else if (有自定义的文字 toolbarDoneBarButtonItemText) {
      用这个 toolbarDoneBarButtonItemText 初始化一个 toobar 右边的 done 按钮
    } else {
      初始化默认的右边的 toolbar 的 done 按钮
    }
    
    if (当前 textfield 没有兄弟 textfield) {
      // 只有一个就不用添加左边的前一个后一个的按钮了，就直接返回
      return
    } else {
      if (有自定义的图片) {
        初始化带图片的 prev 的按钮以及 next 按钮
      } else if (有自定义的文字) {
        初始化带文字的 prev 按钮以及 next 按钮
      } else {
        初始化默认的 prev 按钮以及 next 按钮
      }
    }
    
    if (textfield 实现了 keyboardAppearance 方法) {
      根据 keyboard 的模式设置 toolbar 的样式
    }
    
    if (shouldShowToolbarPlaceholder 属性为 YES) {
      将 textfield 的 placeholder 的样式复制给 toolbar 的 titleBarButton
    } else {
      隐藏 toolbar 的 titleBarButton
    }
    
    if (当前 textfield 是第一个) {
      设置 prev 按钮 enabled 为 NO
    } else if (当前 textfield 是最后一个) {
      设置 next 按钮 enabled 为 NO
    } else {
      设置 prev 和 next 按钮 enabled 为 YES
    }
    // 最后通过设置好的 prev 和 next 和 placeholder 和 done 创建 IQToolbar。并将 toolbar 设置为 textfield 的 inputAccessview
    [textfield setInputAccessView: toolbar];
  }
}
```

代码很简单，就是很繁琐，伪代码都写了这么多。可以看到，IQKeyboardManager 很人性化的为 toobar 上左右的按钮都设置了自定义的图片和文字(虽然一般不会有人去改)

完成按钮的点击事件可以理解为就是取消 textfield 的第一响应者。prev 按钮和 next 按钮实现上稍微复杂一点，但是思想上是非常简单的。以 prev 按钮为例：

```objc
-(void)previousAction:(IQBarButtonItem*)barButton {
  if ([self canGoPrevious]) {
    [self goPrevious];
  }
  ....省略以一部分 firstResponder 转移成功后自定义的处理逻辑(一般不会使用就省略不看了)
}
```

逻辑就是能跳到前面一个就跳到前面一个：

```objc
-(BOOL)canGoPrevious {
  // 获取当前视图的所有兄弟 textfield 
  NSArray<UIView*> *textFields = [self responderViews]
  // 获取当前 textfield 在这些 textfield 的 index
  NSUInteger index = [textFields indexOfObject:_textFieldView];
  // 如果 textfield 的 index 不是第一个
  if (index != NSNotFound && index > 0) {
    return YES;
  } else {
    return NO;
  }
}

-(BOOL)goPrevious {
  // 获取当前视图的所有兄弟 textfield 
  NSArray<__kindof UIView*> *textFields = [self responderViews];
  // 获取当前 textfield 在这些 textfield 的 index
  NSUInteger index = [textFields indexOfObject:_textFieldView];
  // 如果 textfield 的 index 不是第一个
  if (index != NSNotFound && index > 0) {
    UITextField *nextTextField = textFields[index-1];
    UIView *textFieldRetain = _textFieldView;
    // 上一个 textfield 获取焦点
    BOOL isAcceptAsFirstResponder = [nextTextField becomeFirstResponder];
    if (isAcceptAsFirstResponder == NO) {
      [textFieldRetain becomeFirstResponder];
    }
    return isAcceptAsFirstResponder;
  } else {
    return NO;
  }
}
```

有没有很熟悉？和处理 return 键的逻辑类似。拿到所有兄弟 textfield，然后判断当前 textfield 是否是第一个。不是的话就让上一个获取焦点。

说完了添加 IQToolbar，现在再来快速看一下如果 `privateIsEnableAutoToolbar` 为 NO 情况下移除 toolbar

```objc
-(void)removeToolbarIfRequired {
    NSArray<UIView*> *siblings = [self responderViews];
    for (UITextField *textField in siblings) {
        UIView *toolbar = [textField inputAccessoryView];
        if ([textField respondsToSelector:@selector(setInputAccessoryView:)] &&
            ([toolbar isKindOfClass:[IQToolbar class]] && (toolbar.tag == kIQDoneButtonToolbarTag || toolbar.tag == kIQPreviousNextButtonToolbarTag))) {
            textField.inputAccessoryView = nil;
            [textField reloadInputViews];
        }
    }
}
```

移除的方法更简单。直接找到所有兄弟 textfield，如果它们的 inputAccessoryView 是 IQToolbar 类型，那么就清空。

##### 调整视图偏移







## 小技巧

到最后了总结一下看 IQKeyboardManager 源码学到的一些技巧。

### 对一个数组排序：

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

### 通过响应链获得当前视图的 ViewController

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

### 在 load 方法中异步执行初始化操作

```objc
+(void)load
{
    [self performSelectorOnMainThread:@selector(sharedManager) withObject:nil waitUntilDone:NO];
}

```

### 宏定义一个不存在的点

宏定义定义一个不存在的点，然后提供用作初始化。

```objc
#define kIQCGPointInvalid CGPointMake(CGFLOAT_MAX, CGFLOAT_MAX)
```

### 计算方法的执行时间

方法的执行时间可以通过分别获取方法开始执行和执行完毕的时间，然后相减：

```objc
CFTimeInterval startTime = CACurrentMediaTime();
CFTimeInterval elapsedTime = CACurrentMediaTime() - startTime;
```

### iOS 发出输入键盘的声音

iOS 提供了一个方法提供播放键盘声音：

```objc
[[UIDevice currentDevice] playInputClick]
```

比如通讯录选择了首字母可以使用这个方法播放声音。

### 设置随键盘弹出的视图

在没看代码前，你可能会认为 IQKeyboardManager 是通过动画的方式将 IQToolbar 展示在 keyboard 上的。其实 iOS 提供了相关的属性。可以直接将自定义视图设置为 textfield 的 inputAccessoryView 就可以实现效果：

```objc
[textfield setInputAccessView: yourView];
```

