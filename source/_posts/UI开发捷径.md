title: 《iOS UI 开发捷径》读书笔记
date: 2016/7/31 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

读了 《iOS UI 开发捷径》，书上的东西比较简单，也比较零碎。所以做个大致的整理。

<!--more-->

### Interface Builder 概要

- Bundle 主要用来保存可执行代码以及保存资源文件。添加到工程里的资源文件会在编译时复制到 main bundle 中。如果没有，那么在 Copy Bundle Resource 中添加。
- 开发 SDK 的时候，资源文件会单独打包到一个 Bundle 中（静态库中无法添加资源文件）。Bundle 保存在 main bundle 下，所以集成 SDK 的时候要去找对应 Bundle 下的资源。

### 使用 Inferface Builder

- 右键 IB 中的控件拖到代码中自动生成 property 或者 action
- 如果要为 VC 添加一个 xib，需要将 xib 的 view 和 VC 的 view 连线。
- 可以在一个 xib 中创建多个 View，然后通过 `loadNibNamed` 方法获取 View 的数组
- 创建 UIVIewController 的时候，系统会判断是否有同名的 xib 文件。如果有，就直接加载。如果没有就需要调用 `initWithNibNamed` 方法指定 xib 文件名。
- xib 嵌套，不能直接拖个 View 设置其 class。你只能拖一个占位的 UIView，然后用代码将要嵌套的那个视图 addSubView 的方式添加进去。
- 加载 image 有两种方式，一种是通过 `initWithName ` 在 mainBundle 中加载；一种是通过 `initWithContentofFile` 通过路径拼接加载。前者加载的图片会被系统缓存，后者则不会。所以大图片最好用后者。
- 要创建 Bundle 要在 macOS 中找，然后将 Supported Platforms 设置为 iOS
- 只能向 sb 文件中添加 UIViewController，而不能添加 View。
- 一个 sb 可以添加多个 VC，通过 storyboardID 来识别 sb。可以通过 storyboardID 来初始化某个 VC。
- sb 可以通过 segue 来跳转，减少胶水代码
- 使用 sb 就可以省略 window 以及 rootViewController 的创建。
- @UIApplicationMain 会自动生成 main 函数

### 全面学习 xib

- Document → Label 可以设置控件的名称，当然你也可以双击设置。
- Document → Object ID 是对象的唯一标识，当打开 sb 的xml 格式文件的时候，可以看到这个 id。IB 就是通过这个 id 找到不同控件的。
- 控件的属性检查器面板会列出控件及其父类的菜单，UIButton 就会列出 Button，Control，View 三个菜单。
- View 的 `tag` 属性可以标识 View。你可以设置后通过 `viewWithTag(_:)` 找到 View
- View 的 `isUserInteractionEnabled` 标识 View 是否可以交互。
- View 的 `alpha` 会作用于 View 的子视图上。如果想背景半透明，子视图清晰，可以设置 View 的 `backgroundColor` 中的 `alpha`。
- Text 的 `numberOfLines` 表示文本显示行数，0 表示无限制多少行。
- Text 的 `isEnabled` 表示文本是否激活，未激活则显示为灰色
- TextField 的 `clearsOnBeginEditing` 属性决定在编辑时是否清空原有内容
- TextField 的 `returnKeyType` 属性决定右下角返回键的显示样式
- TextField 的 `isSecureTextEntry` 决定是否是密文
- Image 的 `highlightedImage` 设置 ImageView 处于高亮状态下要显示的图片。
- Image 的 `isHightlighted` 属性标识 ImageView 是否处于高亮状态。
- ScrollView 的 `isDirectionLockEnabled` 表示滚动只能左右滚或者上下滚
- ScrollView 的 `keyboardDismissMode` 表示如何收起键盘。
- TableView 的 `separatorInset` 可以设置分隔线的始末位置
- 手势可以像控件一样被托给 View，还可以通过拖动的方式设置手势的 delegate
- TableViewCell 的 `prepareForReuse()` 方法可以用来清除 cell 之前的状态。
- 选中 Assets 中的某张图片，底部有 Show Slicing 按钮，可以设置图片的拉伸状态。
- Asset 中的图片只能用 `init(named:)` 获取

### 在 IB 中使用 Auto Layout

- Auto Layout 中设置里约束也是转化为 frame 的形式。但是约束转化为 frame 是在 layout subViews 阶段完成的，所以再 `viewDidLoad` 中设置 frame 是无效的。要设置必须要在 `viewDidLayoutSubviews` 中进行。
- 如果用代码添加约束，那么要先创建一个约束，然后将这个约束添加到两个 View 的公共父类上。
- 约束存在优先级 `UILayoutPriority`，就是当多个约束有冲突的时候，优先级高的为准。
- 像 Label 这样的属性存在固有尺寸，即没有设置 Width 和 Height 系统会自动通过 text 的属性获得其尺寸。
- 压缩阻力表示一个视图保护器完整性的能力，比如两个 label 并排排列。压缩阻力小的那个被挤压。压缩阻力是针对固有尺寸的，如果设置了 Width 就没有这种情况了。
- 内容吸附表示一个视图能否被拉伸的能力，和压缩阻力正好相反。
- 可以将约束拖到代码中，然后直接设置约束的 `constant` 属性。设置好后一定要加 `self.view.layoutIfNeeded()`，否则不会显示变化。
- 要设置一个 View 的宽高比，右键选择 `aspect ratio`。
- 要设置两个 View 的比例，在一个 View 中右键拖到另一个 View 中，选择 `equal height`。然后设置 `Multiplier` 或者 `constant`

### storyboard 全面学习

- sb 中可以向一个 tableViewController 添加一个或多个 Table View Cell。可以设置其不同的 Identifier。
- Extra View 可以为 VC 提供一个额外的关联视图。你不能将 View 拖到 VC 中，而是应该和 VC 平级。你可以再将这个 View 和 代码关联，在需要的时候添加到 VC 中。
- sb 中的 segue 用 Identifier 区分。Class 是 segue 的类型，一般为 UIStoryboardSegue
- 可以重写 `prepare(for:sender:)` 方法。通过 Identifier 找到对应的 segue，从而确定跳转的页面关系，`segue.destination` 就是目标 VC。
- 可以重写 `shouldPerformSegue(withIdentifier:sender:)` 来判断是否需要跳转。
- 可以调用 `performSegue(withIdentifier:sender:)` 来自己触发跳转

### IB 进阶

- 针对不同的机器，可以选择某一个约束，然后点击 Constant 旁边的 “+”，增加针对某个屏幕的约束
- User Defined Runtime Atrributes 可以在运行的时候设置属性，但是仅限几个基本类型。比如 `layer.borderColor` 的类型为 CGColor，但是 User Defined Runtime Attributes 只能添支持 UIColor 的属性，所以直接这样就不行。你可以自己设置一个 `borderColor` 属性，然后在 extension 中添加这个属性的读写方法即可。
- 通过 Storyboard Reference 连接多个 sb。还是像拖控件一样找到 StoryBoard Reference 然后拖出来。
- 可以像拖控件一样，拖出 Object。我们不能将其拖到 VC 上，它和 VC 同级别。我们可以设置其 Class，作为 VC 中的 View 的 delegate。
- @IBInspectable 写在 var 前面：`@IBInspectable var bigColor: UIColor = UIColor.red`。这样就可以通过 sb 的面板里找到 bigColor 直接设置 `UIColor.red`。支持的属性包括几个基本属性，表示位置信息的几个类型(CGPoint 等)，UIColor，UIImage。反正就是一些平时能设置的东西。


- 选中某个 View，然后按住 option，就可以获得这个 View 与其他 View 的距离关系了。

  ​