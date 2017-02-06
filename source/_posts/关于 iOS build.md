title: 关于 iOS build
date: 2017/2/3 10:07:12  
categories: iOS
tags:
	- Xcode

---

关于 Xcode 是如何编译的，下面是一些较为浅显的讨论。

<!--more-->

## 点击run后发生了什么
点击 Run 之后 App 进行**编译、汇编、链接、代码签名以及启动执行**等操作

### 编译
Clang是LLVM的前端，可以用来编译C，C++，ObjectiveC等语言。传统的编译器通常分为三个部分，**前端(frontEnd)**，**优化器(Optimizer)**和**后端(backEnd)**。在编译过程中，前端主要负责词法和语法分析，将源代码转化为抽象语法树；优化器则是在前端的基础上，对得到的中间代码进行优化，使代码更加高效；后端则是将已经优化的**中间代码**转化为针对各自平台的机器代码。Clang则是以LLVM为后端的一款高效易用，并且与IDE结合很好的编译前端。

![编译器解构](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build_编译器.png?raw=true)

编译器分别编译器前端（clang）和编译器后端，编译器前端负责产生机器无关的中间代码，编译器后端负责对中间代码进行优化并转化为目标机器代码，对于为什么需要 中间代码这个东西，看个图就一目了然啦（IR：intermediate representation中间表示）

![中间代码](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build_中间代码.png?raw=true)


### 汇编
目标代码需要经过汇编器处理，才能变成机器上可以执行的指令。生成对应的`.o`文件

### 链接
链接器（这里指的是静态链接器）将多个目标文件合并为一个可执行文件，在 OS X 和 iOS中的可执行文件是 `Mach-O` 。链接呢，又分为静态链接和动态链接

静态链接：在编译链接期间发挥作用，把目标文件和静态库一起链接形成可执行文件。

![链接过程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build_链接过程.png?raw=true)

动态链接：链接过程推迟到运行时再进行。对于动态链接和静态链接，各有千秋。

### 代码签名
我们每次 `build` 之后，都会发现工程目录下多了一个 `.app` 文件。

![代码签名](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build_代码签名.png?raw=true)

在 `.app` 目录中，有又一个叫 `_CodeSignature` 的子目录，这是一个  `plist` 文件，里面包含了程序的代码签名，你的程序一旦签名，就没有办法更改其中的任何东西，包括资源文件，可执行文件等，iOS系统会检查这个签名。

签名过程本身是由命令行工具 `codesign` 来完成的。如果你在 Xcode 中build 一个应用，这个应用构建完成之后会自动调用 `codesign` 命令进行签名，这也是 Link 之后的一个关键步骤。

### 启动
在启动过程中，`dyld`（动态链接器） 起了很重要的作用，进行动态链接，进行符号和地址的一个绑定。

`dyld` 主要在启动过程中主要做了以下事情：
- 加载所依赖的 `dylibs`
* `Fix-ups`：Rebase 修正地址偏移，因为 OS X和 iOS 通过 `ASLR` 来做地址偏移（随机化）来避免收到攻击
* `Fix-ups`：Binding确定 `Non-Lazy Pointer` 地址，进行符号地址绑定。
* `ObjC runtime` 初始化：加载所有类
* `Initializers` ：执行 `load` 方法和 `__attribute__((constructor))` 修饰的函数

## Build过程
在选择工程后会出现下面这样的几个选项：
![build过程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build过程.png?raw=true)

### Build Phases

Build Phases 代表着将代码转变为可执行文件的最高级别规则。里面描述了 build 过程中必须执行的不同类型规则。

![build phases](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/build_build过程.png?raw=true)

首先是 target 依赖项的构建。这里会告诉 build 系统，build 当前的 target 之前，必须先对这里的依赖性进行 build。实际上这并不属于真正的 build phase，在这里，Xcode 只不过将其与 build phase 显示到一块罢了。

接着在 build phase中是一个 CocoaPods 相关的脚本 *script execution* 。接着在 Compile Sources section 中规定了所有必须参与编译的文件。此处列出的所有文件将根据相关的 rules 和 settings 被处理。

当编译结束之后，接下来就是将编译所生成的目标文件链接到一块。注意观察，Xcode 中的 build phase 之后是："Link Binary with Libraries." 这里面列出了所有的静态库和动态库，这些库会参与上面编译阶段生成的目标文件进行链接。

当链接完成之后，build phase 中最后需要处理的就是将静态资源（例如图片和字体）拷贝到 app bundle 中。需要注意的是，如果图片资源是PNG格式，那么不仅仅对其进行拷贝，还会做一些优化。

虽然静态资源的拷贝是 build phase 中的最后一步，但 build 还没有完成。例如，还没有进行 code signing （这并不是 build phase 考虑的范畴），code signing 属于 build 步骤中的最后一步 "Packaging"。

### Build Rules

Build rules 指定了不同的文件类型该如何编译。一般来说，开发者并不需要修改这里面的内容。如果你需要对特定类型的文件添加处理方法，那么可以在此处添加一条新的规则。一条 build rule 指定了其应用于哪种类型文件，该类型文件是如何被处理的，以及输出的内容该如何处置。

### Build Settings

至此，我们已经了解到在 build phases 中是如何定义 build 处理的过程，以及 build rules 是如何指定哪些文件类型在编译阶段需要被预处理。在 build settings 中，我们可以配置每个任务的详细内容。

你会发现 build 过程的每一个阶段，都有许多选项：从编译、链接一直到 code signing 和 packaging。注意，settings 是如何被分割为不同的部分 -- 其实这大部分会与 build phases 有关联，有时候也会指定编译的文件类型。

### 工程文件

上面我们介绍的所有内容都被保存在工程文件（`.pbxproj`）中，除了其它一些工程相关信息（例如 file groups），我们很少会深入该文件内部，除非在代码 merge 时发生冲突，或许会进去看看。

如果用文本编辑器打开一个工程文件，从头到尾看一遍里面的内容。它的可读性非常高，里面的许多内容一看就知道什么意思了，不会存在太大的问题。通过阅读并完全理解工程文件，这对于合并工程文件的冲突非常有帮助。

首先，我们来看看文件中叫做 rootObject 的条目。在我的工程中，如下所示：

```ruby
rootObject = 1793817C17A9421F0078255E /* Project object */;
```

根据这个 ID（1793817C17A9421F0078255E），我们可以找到 main 工程的定义：

```ruby
/* Begin PBXProject section */
    1793817C17A9421F0078255E /* Project object */ = {
        isa = PBXProject;
...
```

在这部分中有一些 keys，顺从这些 key，我们可以了解到更多关于这个工程文件的组成。例如，mainGroup 指向了 root file group。如果你按照这个思路，你可以快速了解到在 `.pbxproj` 文件中工程的结构。

```ruby
		1BC1E1511ACBEFFA0042431B = {
			isa = PBXGroup;
			children = (
				B62C765C1D9B71D300CB2CD4 /* OKit.framework */,
				230935DA1C4F3D9C0070A5B3 /* QiNiuManager.xcodeproj */,
				23D1991E1C2B8FF600E1DE8A /* Debug.xcodeproj */,
				0DD10D591C28FB4900ADF8E5 /* Track.xcodeproj */,
				0D106DDC1BE70BF600C49E7B /* Interface.xcodeproj */,
				0D106D761BE7052800C49E7B /* Model.xcodeproj */,
				0DC641041C0D427A007025D5 /* Wallet.xcodeproj */,
				0DC641A41C0D997C007025D5 /* Tool.xcodeproj */,
				0DC641CA1C0D9ADB007025D5 /* Location.xcodeproj */,
				1BC1E1631ACBEFFA0042431B /* Mike */,
				1BC1E15C1ACBEFFA0042431B /* Frameworks */,
				1BC1E15B1ACBEFFA0042431B /* Products */,
				FF0DD481DA78D64E15039700 /* Pods */,
			);
			sourceTree = "<group>";
		};
```

下面来看一些与 build 过程相关的内容。其中 target key 指向了 build target 的定义：

```ruby
			targets = (
				1BC1E1591ACBEFFA0042431B /* Mike */,
			);
```

根据第一个内容，我们找到一个 target 的定义：

```ruby
1BC1E1591ACBEFFA0042431B /* Mike */ = {
			isa = PBXNativeTarget;
			buildConfigurationList = 1BC1E1861ACBEFFA0042431B /* Build configuration list for PBXNativeTarget "Mike" */;
			buildPhases = (
				E2E1BAED1E12657500A4F13C /* ShellScript */,
				D328411202E8C8A158C68A9A /* [CP] Check Pods Manifest.lock */,
				32F9C08DAB34C3D77A8CD59E /* [CP] Check Pods Manifest.lock */,
				1BC1E1561ACBEFFA0042431B /* Sources */,
				1BC1E1571ACBEFFA0042431B /* Frameworks */,
				1BC1E1581ACBEFFA0042431B /* Resources */,
				7C20CC96F9690C1C11011773 /* [CP] Embed Pods Frameworks */,
				1DBEA7A742BF77DA6EC31CEB /* [CP] Copy Pods Resources */,
				2337E6001BCDF5EC007659ED /* ShellScript */,
				2CA588C55C8E11D9779AABF0 /* 📦 Embed Pods Frameworks */,
				789C695793C24ECAC7D8420E /* 📦 Copy Pods Resources */,
				B60684181D9924B00003E3E5 /* Embed Frameworks */,
			);
			buildRules = (
			);
			dependencies = (
				230935E91C4F55B20070A5B3 /* PBXTargetDependency */,
				23D199251C2B918700E1DE8A /* PBXTargetDependency */,
				0DD10D601C28FB5D00ADF8E5 /* PBXTargetDependency */,
				0DE3C0941C0F103000184463 /* PBXTargetDependency */,
				0DE3C0921C0F102D00184463 /* PBXTargetDependency */,
				2390C7521C0DB98B008C31B1 /* PBXTargetDependency */,
				0D106DE41BE70C2600C49E7B /* PBXTargetDependency */,
				0D106D7E1BE7060100C49E7B /* PBXTargetDependency */,
			);
			name = Mike;
			productName = Mike;
			productReference = 1BC1E15A1ACBEFFA0042431B /* Mike.app */;
			productType = "com.apple.product-type.application";
		};
```

其中 `buildConfigurationList` 指向了可用的配置项，一般是 Debug 和 Release。

```ruby
		1BC1E1861ACBEFFA0042431B /* Build configuration list for PBXNativeTarget "Mike" */ = {
			isa = XCConfigurationList;
			buildConfigurations = (
				1BC1E1871ACBEFFA0042431B /* Debug */,
				1BC1E1881ACBEFFA0042431B /* Release */,
			);
			defaultConfigurationIsVisible = 0;
			defaultConfigurationName = Release;
		};
```

根据 debug 对应的 id，我们可以找到 build setting tab 中所有选项存储的位置：

```ruby
1BC1E1871ACBEFFA0042431B /* Debug */ = {
			isa = XCBuildConfiguration;
			baseConfigurationReference = 31FD03C25841C700BD562061 /* Pods-Mike.debug.xcconfig */;
			buildSettings = {
              	ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon;
				ASSETCATALOG_COMPILER_LAUNCHIMAGE_NAME = LaunchImage;
				CLANG_CXX_LANGUAGE_STANDARD = "compiler-default";
				CLANG_CXX_LIBRARY = "compiler-default";
				CLANG_ENABLE_OBJC_ARC = YES;
              ...
```

`buildPhases` 属性则简单的列出了在 Xcode 中定义的所有 `build phases`。这非常容易识别出来（Xcode 中的参数使用了它们原本真正的名字，并以 C 风格进行注释）。`buildRules` 属性是空的：因为在该工程中，我没有自定义 `build rules`。`dependencies` 列出了在 `Xcode build phase tab` 中列出的 target 依赖项。





