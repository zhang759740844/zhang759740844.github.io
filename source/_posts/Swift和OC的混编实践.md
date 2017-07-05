title: Swift 和 OC 的混编实践
date: 2017/7/5 10:07:12  
categories: iOS
tags:
	- Cocoapods
	- Swift
---

最近想用 Swift 重写项目，接触了一下 OC 和 Swift 的混编技巧，发现坑还是挺多的。尤其是使用 cocoapods 导入的库的混编。这里记录一下各种应用场景。

<!--more-->

## 主工程中使用

### Swift 中调用 OC

首先创建一个 OC 的项目，然后直接在工程里添加 Swift 文件。此时会弹出一个提示，问你是否添加一个桥接文件，你需要自己添加要暴露给 Swift 使用的 OC 的头文件。

自动帮你创建的，不要白不要啊，所以果断点击 create。

![添加桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCWithSwift1.png?raw=true) 

这个时候就会生成一个 `项目名-Bridgeing-Header.h` 的头文件：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCWithSwift2.png?raw=true)

当然，你也可以自己创建与设置，在 Build Settings 中设置：

![设置桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCWithSwift3.png?raw=true)

现在在桥接文件中添加要暴露给 Swift 调用的头文件：

![添加文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCWithSwift4.png?raw=true)

然后在 Swift 中调用即可，非常的 easy：

![调用](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCWithSwift5.png?raw=true)



### OC 中调用 Swift

OC 中要调用 Swift 也要导入一个头文件，这个头文件对外不可显示，其命名方式是：`模块名-Swift.h`。可以在 Build setting 中查看：

![查看头文件命名](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftWithOC1.png?raw=true)

现在我们在 OC 中添加这个头文件：

![添加头文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftWithOC2.png?raw=true)

比较 OC 和 Swift 的调用方式还是不同的，我们需要查看一下到底可以调用那些方法和类。这个时候，我们可以点进这个头文件中去。进入头文件往下拉，可以看到暴露出来的 Swift 的类名和方法：

![头文件中的类](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftWithOC3.png?raw=true)



## 涉及 cocoapods

上面完全是主工程里使用当然毫无难度，但是日常的开发中肯定要用到其他的别人的库，或者自建的私有库，这个时候怎么混编呢？我们先用 `pod lib create OCWithSwift` 命令创建一个类库。然后在其中分别添加一个 Swift 文件和一个 OC 文件：

![添加文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib1.png?raw=true)

### 库中 Swift 调用 OC

库中 Swift 调用 OC 和主工程中有什么不同呢？我们先到 Build Settings 中查看这个头文件应该叫什么：

![头文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib2.png?raw=true)

发现什么也没有。这个时候千万不要手贱去创建一个，和主工程中不同的是，使用cocoapods 创建的库里面我们不需要自己创建这个桥接文件了。现在直接在 Swift 文件中调用，可以使用 OC 的类和方法：

![创建头文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib3.png?raw=true)

我们点进这个类中，发现 Xcode 已经自动生成了这个 OC 文件的 Swift 代码：

![生成的Swift代码](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib4.png?raw=true)

那么这是为什么呢？搜索一下我们可以发现在 `OCWithSwift-umbrella.h` 中引用了这个 OC 文件。所以这个头文件其实就是做了桥接的作用，应该是由 cocoapods 设置的：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib5.png?raw=true)

为了测试，我们再创建一个 OC 文件 `OCFile_2.h`，然后同样导入到 `OCWithSwift-umbrella.h`:

![添加新文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib6.png?raw=true)

这个时候编译，报出异常：

![异常](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib7.png?raw=true)

我们需要做的就是将这个文件设置为 Public：

![添加为 Public](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib8.png?raw=true)

现在再编译就没有问题了。可以直接在 Swift 中使用：

![使用](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftOCLib9.png?raw=true)

### 库中 OC 调动 Swift

库中 OC 调用 Swift 的过程还是大致相同的。先看看 Build Settings：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib1.png?raw=true)

我们在 OC 文件中引入，不过这次要引入的格式为 `#import <产品名/模块名-Swift.h>`，我们还是可以在 Build Settings 中查看：

![名称](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib2.png?raw=true)

现在在 OC 中使用，编译后报出错误，`use of undeclared identifier 'SwiftFile'`:

![错误](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib3.png?raw=true)

为什么 `SwiftFile` 会是未被声明的呢？我们进入 `OCWithSwift-Swift.h` 中查看：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib4.png?raw=true)

桥接文件中并没有发现 `SwiftFile` 的身影，确实是没有声明。那么要如何才能将 `SwiftFile` 暴露出来呢？为 Swift 类和方法添加 public 修饰符：

![添加修饰符](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib5.png?raw=true)

重新 Build 一下， 我们再次进入 `OCWithSwift-Swift.h` 文件，方法出现了：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCSwiftLib6.png?raw=true)

### 主工程中 Swift 调用库中类与方法

在上一步中，其实库中的 Swift 和 OC 都已经生成了对应的 OC 和 Swift 方法。在主工程中直接引入调用即可:

![使用](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SwiftLibOC1.png?raw=true)

可以看到，不论是 Swift 直接调用 Swift，还是 Swift 调用 OC 转换后的 Swift 都是没有问题的。

### 主工程中 OC 调用库中类与方法

同样在主工程中的 OC 类中引用需要的类即可：

![桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/OCLibSwift1.png?raw=true)

OC 调用库中 OC 和 Swift 转换的 OC 同样没有问题。



到这里基本上该趟的坑都趟过了

[参考自官方文档](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html#//apple_ref/doc/uid/TP40014216-CH10-XID_87)

[我的demo](https://github.com/zhang759740844/MyOCDemo/tree/develop/OCWithSwift)



