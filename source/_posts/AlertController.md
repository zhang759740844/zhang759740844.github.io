title: UIAlertController 使用简介
date: 2017/2/7 14:07:12  
categories: iOS
tags: 

	- 基本控件
---

`UIAlertController` 是 iOS 8 中推出的新特性，用以代替 `UIAlertView` 和 `UIActionSheet`。下面学习一下它的 api 的使用。内容很基础，只是当做备忘。

<!--more-->

`UIAlertController` 定义了一个枚举类型用以区分 `UIAlertView` 和 `UIActionSheet`：

```objc
typedef NS_ENUM(NSInteger, UIAlertControllerStyle) {
    UIAlertControllerStyleActionSheet = 0,
    UIAlertControllerStyleAlert
} NS_ENUM_AVAILABLE_IOS(8_0);
```

### API

列举一下在 `UIAlertController` 中使用的 API：

```objc
//Create UIAlertController
+ (instancetype)actionWithTitle:(NSString *)title style:(UIAlertActionStyle)style handler:(void (^)(UIAlertAction *action))handler;

//Adding an action
- (void)addAction:(UIAlertAction *)action;
  
//Adding text filed to UIAlertController.This method is supported only for  UIAlertControllerStyleAlert
- (void)addTextFieldWithConfigurationHandler:(void (^)(UITextField *textField))configurationHandler;
```

### 创建一个基本的对话框

```objc
    UIAlertController * alert=   [UIAlertController alertControllerWithTitle:@"My Title"
                                                                     message:@"Enter User Credentials"
                                                              preferredStyle:UIAlertControllerStyleAlert];
    [self presentViewController:alert animated:YES completion:nil];
```

很容易理解的方法调用。由于是一个 `Controller`，所以使用 `presentViewController` 方法将 View 显示出来。此时的 View 还没有点击按钮。

### 创建一个带两个选项按钮的对话框

在上面的基础上，添加 OK/Cancel 按钮，并设置点击方法。这就需要我们设置一个 `action` ，然后把它添加到 `UIAlertController` 中。这里只演示了一下 OK 的情况，Cancel 同理。记得和上面合并的时候要把 `presentViewController:animated:completion:` 方法写在最后啊。

```objc
    UIAlertAction* ok = [UIAlertAction actionWithTitle:@"OK"
                                                 style:UIAlertActionStyleDefault
                                               handler:^(UIAlertAction * action){
                                                   //Do some thing here
                                                   [alert dismissViewControllerAnimated:YES completion:nil];
                                               }];
    [alert addAction:ok]; // add action to uialertcontroller
```

这里面的 `style` 是一个枚举类型：

```objc
typedef NS_ENUM(NSInteger, UIAlertActionStyle) {
    UIAlertActionStyleDefault = 0,		
    UIAlertActionStyleCancel,			
    UIAlertActionStyleDestructive		
} NS_ENUM_AVAILABLE_IOS(8_0);
```

在 `AlertView` 中的 `UIAlertActionStyleCancel` 是红色，`UIAlertActionStyleDestructive` 是加粗。

### 创建一个带 textInput 的对话框

```objc
[alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
	textField.placeholder = @"Username";
}];

[alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
    textField.placeholder = @"Password";
    textField.secureTextEntry = YES;
}];
```

添加了一个输入框和一个密码输入框。可以设置 `textField` 的一些属性。

### 创建一个 ActionSheet

```objc
    UIAlertController * view=   [UIAlertController alertControllerWithTitle:@"My Title"
                                                                    message:@"Select you Choice"
                                                             preferredStyle:UIAlertControllerStyleActionSheet];
    
    UIAlertAction* ok = [UIAlertAction actionWithTitle:@"OK"
                                                 style:UIAlertActionStyleDefault
                                               handler:^(UIAlertAction * action){
                             //Do some thing here
                                                   [view dismissViewControllerAnimated:YES completion:nil];
                                               }];
    UIAlertAction* cancel = [UIAlertAction actionWithTitle:@"Cancel"
                                                     style:UIAlertActionStyleDefault
                                                   handler:^(UIAlertAction * action){
                                                       [view dismissViewControllerAnimated:YES completion:nil];
                                                   }];
    [view addAction:ok];
    [view addAction:cancel];
    [self presentViewController:view animated:YES completion:nil];
```

这里的 `style` 如果是 `UIUIAlertActionStyleCancel` 那么就会自动放在最底下；如果是 `UIAlertActionStyleDestructive` 就会变成红色。

### UIAlertView 和 UIAlertController 的区别

- `UIAlertView` :iOS 系统为了保证 `UIAlertView` 在所有界面之上，它会临时创建一个新的 `UIWindow`，通过将 `UIWindow` 的 `UIWindowLevel` 设置的更高，让 `UIAlertView` 盖在所有应用的界面之上。
- `UIAlertController`:继承自 `UIViewController`，它采用一个 `UIPopoverPresentationController` 类进行管理，`UIPopoverPresentationController` 又继承自 `UIPresentationController`，其中的`presentingViewController` 属性表示展示之前的 Controller，`presentedViewController` 属性表示被展示的Controller。另外这种方式，也统一了iPhone和iPad的使用方式

### 自定义的 UIAlertController

如果系统的提示框不能满足业务需求，就需要自定义。最方便的方式还是在当前的 UIWindow 上添加视图。不过我们还是以 ViewController 的方式创建。

但是当你把 UIViewController 的背景色设为透明，然后 present 的时候，会发现页面是黑色的。这是因为 present 出来的 viewController 表面上看起来是盖在了原有界面的上面，但那其实只是个动画，最终实际上是替换掉了，所以设置透明以后会导致下面没有东西所以就变成黑色了。

我们需要设置 ViewController 的 `modalPresentationStyle` 属性为 `UIModalPresentationOverCurrentContext`，表示 present 出来的 ViewController 是盖在之前的 ViewController 上的。**注意，不能再 `viewDidload` 中设置, 必须在初始化的时候设置好,然后再 present**：

```objc
MyAlertViewController *alertViewController = [MyAlertViewController alloc] init];
modalPresentationStyle = UIModalPresentationOverCurrentContext;
[self presentViewController: alertViewController animated: false];
```

这种方式禁掉了动画，如果需要过渡效果，可以自定义 ViewController 的 modal 动画实现。