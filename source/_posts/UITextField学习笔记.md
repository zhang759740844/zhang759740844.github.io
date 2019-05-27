title: UITextField 简介
date: 2017/4/23 14:07:12  
categories: iOS
tags: 

	- 基本控件
---

`UITextFied` 是经常用到的一个控件。还是有一些点需要注意的。 

<!--more-->

### 基本属性

挑选一些可能会忘记需要查阅的常用的基本属性，可能不完全，比如背景色啊，占位符啊，设置和读取文字啊都没列入。

#### 设置边框样式

```objc
textField.borderStyle = UITextBorderStyleRoundedRect;//圆角

//可选属性：
UITextBorderStyleNone,无边框
UITextBorderStyleLine,有边框
UITextBorderStyleBezel,有边框和阴影
UITextBorderStyleRoundedRect圆角
```

#### 设置字体

```objc
[textField setFont:[UIFont fontWithName:@"Arial" size:30]];
```

#### 密文输入

```objc
textField.secureTextEntry = YES; 
```

#### 键盘类型

```objc
textField.keyboardType = UIKeyboardTypeNumberPad;  数字键

//全部类型    
UIKeyboardTypeDefault,当前键盘（默认）
UIKeyboardTypeASCIICapable,字母输入键
UIKeyboardTypeNumbersAndPunctuation,数字和符号
UIKeyboardTypeURL,URL键盘
UIKeyboardTypeNumberPad,数字键盘
UIKeyboardTypePhonePad,电话号码输入键盘   
UIKeyboardTypeEmailAddress,邮件地址输入键盘
```

#### 键盘风格

```objc
textView.keyboardAppearance=UIKeyboardAppearanceDefault；

//可选属性
UIKeyboardAppearanceDefault， 默认外观，浅灰色
UIKeyboardAppearanceAlert，     深灰 石墨色
```

#### 设置左右视图

`textField` 可以 `leftView` 和 `rightView`，文本输入区域在两者之间。设置清除按钮样式

```objc
textField.clearButtonMode = UITextFieldViewModeAlways;
//Mode同左右视图的mode一样。
```

#### 再次编辑时是否清除之前内容

```objc
// 默认是NO
textField.clearsOnBeginEditing = YES
```

#### 对齐方式

垂直对齐：

```objc
textField.contentVerticalAlignment = UIControlContentVerticalAlignmentCenter 

//可选属性：
UIControlContentVerticalAlignmentCenter  居中对齐
UIControlContentVerticalAlignmentTop    顶部对齐，默认是顶部对齐
UIControlContentVerticalAlignmentBottom 底部对齐
UIControlContentVerticalAlignmentFill    完全填充
```

水平对齐：

```objc
textField.textAlignment = UITextAlignmentCenter;

//可选属性：
UITextAlignmentLeft，左对齐，默认是左对齐
UITextAlignmentCenter，
UITextAlignmentRight，右对齐 
```

#### 设置滚动

```objc
textField.adjustsFontSizeToFitWidth = YES; 
```

默认是 NO。当是  YES当充满边框时，文字会缩小，当小到一定程度时仍然会滚动。

```objc
textField.minimumFontSize = 20; 
```

设置滚动时候的最小字号。

#### 设置return键

```objc
textField.returnKeyType = UIReturnKeyGoogle;search

//可选属性
UIReturnKeyDefault, 默认 灰色按钮，标有Return
UIReturnKeyGo,标有Go的蓝色按钮
UIReturnKeyGoogle,标有Google的蓝色按钮，用语搜索
UIReturnKeyJoin,标有Join的蓝色按钮
UIReturnKeyNext,标有Next的蓝色按钮
UIReturnKeyRoute,标有Route的蓝色按钮
UIReturnKeySearch,标有Search的蓝色按钮
UIReturnKeySend,标有Send的蓝色按钮
UIReturnKeyYahoo,标有Yahoo的蓝色按钮
UIReturnKeyYahoo,标有Yahoo的蓝色按钮
UIReturnKeyEmergencyCall, 紧急呼叫按钮
```



### 文本编辑框代理

首先要注意，必须先设置当前 textField 的 `delegate`，否则下面的回调无法响应。可以代码设置，也可以在 xib 中连当前 textField 的 `delegate` 到 `file's owner` 上。

#### 是否进入编辑模式

```objc
- (BOOL)textFieldShouldBeginEditing:(UITextField *)textField
```

默认返回YES，进入编辑模式。NO不进入编辑模式。

#### 进入编辑模式

```objc
- (void)textFieldDidBeginEditing:(UITextField *)textField
```

#### 是否退出编辑模式

```objc
- (BOOL)textFieldShouldEndEditing:(UITextField *)textField
```

默认返回YES，退出编辑模式。NO不退出编辑模式

#### 退出编辑模式

```objc
- (void)textFieldDidEndEditing:(UITextField *)textField
```

#### 点击清除按钮是否清空

```objc
- (BOOL)textFieldShouldClear:(UITextField *)textField
```

默认返回YES，返回NO不清除。

#### 点击键盘上Return按钮时候调用

```objc
- (BOOL)textFieldShouldReturn:(UITextField *)textField
```

#### 输入任何字符的时候调用该方法

```objc
-(BOOL)textField:(UITextField *)field shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string
```

当输入字符时，代理调用该方法，如果返回YES则这次输入可以成功，如果返回NO，不能输入成功。range表示光标位置，string表示这次输入的字符串。

### 一些技巧

#### 全屏触摸关闭

监听手势，当点击开始时，关闭 textfield

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [self.textField resignFirstResponder];
}
```

如果你不能拿到 textfield 的实例，其实可以直接设置 `endEditing` 属性，表示结束编辑：

```objc
[self.view endEditing:YES];
```

#### 限制长度

使用上面的 `shouldChangeCharactersInRange:` 方法：

```objc
- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string{  
  	//string就是此时输入的那个字符textField就是此时正在输入的那个输入框返回YES就是可以改变输入框的值NO相反
	//按会车可以改变
  	if ([string isEqualToString:@"\n"]){ 
        return YES; 
    } 

    NSString * toBeString = [textField.text stringByReplacingCharactersInRange:range withString:string]; //得到输入框的内容
	//判断是否时我们想要限定的那个输入框
    if (self.myTextField == textField){ 
        if ([toBeString length] > 20) { //如果输入框内容大于20则弹出警告
            textField.text = [toBeString substringToIndex:20]; 
            UIAlertView *alert = [[[UIAlertView alloc] initWithTitle:nil message:@"超过最大字数不能输入了" delegate:nil cancelButtonTitle:@"Ok" otherButtonTitles:nil, nil] autorelease]; 
            [alert show]; 
            return NO; 
        } 
    } 
    return YES; 
}
```

 

#### UITextField输入中文时文字下沉的解决方法

出现情况：在 xib 创建的 `UITextField` 中输入中文的时候，`Border Style` 这个属性设置为第一种 `UITextBorderStyleNone`，同时 `Clear Button` 设置为 `Never appears` 时会出现这种情况

解决方法：可以设置 `Clear Button` 的出现时机，或者设置 `Border Style` 为其他类型 。如果一定要是这种 `Border Style` 和 `Clear Button`，那么可以将 xib 中的 `UITextField` 拖出来，通过代码设置 `Border Style` 的方式设置其为`UITextBorderStyleNone` (在 xib 中要先设置为其他的再在代码里这样设置才能成功)。

#### 多个 UITextField 点击换行按钮，切换 FirstResponder

主要就是利用 `textFieldShouldReturn:` 这个回调方法：

```objc
- (BOOL)textFieldShouldReturn:(UITextField *)textField{
    if (textField == errorTF1) {
        [errorTF2 becomeFirstResponder];
    } else if (textField == errorTF2) {
        [errorTF3 becomeFirstResponder];
    } else if (textField == errorTF3) {
        [errorTF4 becomeFirstResponder];
    } else if (textField == errorTF4){
        [errorTF4 resignFirstResponder];
    }

    return NO;//YES or NO,that is a question
}
```



#### textField 键盘遮挡界面上移

主要的思想就是监听键盘弹出和消失的通知，然后判断是否需要将屏幕上移。完整代码如下：

```objc
- (void)viewDidLoad  {  
    [super viewDidLoad];  
    //Do any additional setup after loading the view, typically from a nib.  
    //self.view.backgroundColor = [UIColor underPageBackgroundColor];  
    myTextField = [[UITextField alloc] init];//初始化UITextField  
    myTextField.frame = CGRectMake(35, 230, 250, 35);  
    myTextField.delegate = self;//设置代理  
    myTextField.borderStyle = UITextBorderStyleRoundedRect;  
    myTextField.contentVerticalAlignment = UIControlContentVerticalAlignmentCenter;//垂直居中  
    myTextField.placeholder = @"Please entry your content!";//内容为空时默认文字  
    myTextField.returnKeyType = UIReturnKeyDone;//设置放回按钮的样式  
    myTextField.keyboardType = UIKeyboardTypeNumbersAndPunctuation;//设置键盘样式为数字  
    [self.view addSubview:myTextField];  
      
      
    //注册键盘出现与隐藏时候的通知  
    [[NSNotificationCenter defaultCenter] addObserver:self  
                                             selector:@selector(keyboadWillShow:)  
                                             name:UIKeyboardWillShowNotification  
                                             object:nil];  
    [[NSNotificationCenter defaultCenter] addObserver:self  
                                             selector:@selector(keyboardWillHide:)  
                                          name:UIKeyboardWillHideNotification  
                                          object:nil];  
    //添加手势，点击屏幕其他区域关闭键盘的操作  
    UITapGestureRecognizer *gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(hideKeyboard)];  
    gesture.numberOfTapsRequired = 1;//手势敲击的次数  
    [self.view addGestureRecognizer:gesture];  
}  
  
//键盘出现时候调用的事件  
-(void) keyboadWillShow:(NSNotification *)note{  
    NSDictionary *info = [note userInfo];  
    CGSize keyboardSize = [[info objectForKey:UIKeyboardFrameEndUserInfoKey] CGRectValue].size;//键盘的frame  
    CGFloat offY = (460-keyboardSize.height)-myTextField.frame.size.height;//屏幕总高度-键盘高度-UITextField高度  
    [UIView beginAnimations:nil context:NULL];//此处添加动画，使之变化平滑一点  
    [UIView setAnimationDuration:0.3];//设置动画时间 秒为单位  
    myTextField.frame = CGRectMake(35, offY, 250, 35);//UITextField位置的y坐标移动到offY  
    [UIView commitAnimations];//开始动画效果  
      
}  
//键盘消失时候调用的事件  
-(void)keyboardWillHide:(NSNotification *)note{  
    [UIView beginAnimations:nil context:NULL];//此处添加动画，使之变化平滑一点  
    [UIView setAnimationDuration:0.3];  
    myTextField.frame = CGRectMake(35, 230, 250, 35);//UITextField位置复原  
  
    [UIView commitAnimations];  
}  
//隐藏键盘方法  
-(void)hideKeyboard{  
    [myTextField resignFirstResponder];  
}  
#pragma mark -  
#pragma mark UITextFieldDelegate  
//开始编辑：  
- (BOOL)textFieldShouldBeginEditing:(UITextField *)textField  {  
    return YES;  
}  
  
//点击return按钮所做的动作：  
- (BOOL)textFieldShouldReturn:(UITextField *)textField  {  
    [textField resignFirstResponder];//取消第一响应  
    return YES;  
}  
  
//编辑完成：  
- (void)textFieldDidEndEditing:(UITextField *)textField  {  
}  
  
-(void)viewDidDisappear:(BOOL)animated{  
    [super viewDidDisappear:animated];  
    [[NSNotificationCenter defaultCenter] removeObserver:self];//移除观察者  
}  
```

#### 设置 UITextField 的 Padding（leftview 和 rightview）

当设置 `UITextField` 为左对齐的时候，就会发现输入的字和 `placeholder` 会紧贴着左边框。想要将字往右移，可以利用好 `leftView` 和 `rightView` 这两个属性。

这两个属性会被放在 `textField` 的最左边和最右边。我们只要创建一个 `UIView` 然后赋值给 `leftView` 就可以了：

```objc
UIView spacerView = [[UIView alloc] initWithFrame: CGRectMake(0, 0, 10, 10)]; 
[textfield setLeftViewMode: UITextFieldViewModeAlways]; 
[textfield setLeftView: spacerView];
```

注意这里要将 mode 设置为 `UITextFieldViewModeAlways`，否则左视图无法显现出来。

#### 点击事件，不触发弹出键盘，触发其他事件

就是不让直接修改，点击后触发自己的响应。只要实现代理方法就可以了：

```objc
- (BOOL)textFieldShouldBeginEditing:(UITextField *)textField{
//写你要实现的：页面跳转的相关代码
        return NO；
}
```

先完成自己的操作，然后设置这个 `textField` 不能响应。




