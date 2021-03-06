title: iOS开发工具应用中的坑
date: 2016/9/23 10:07:12  
categories: iOS
tags:
	- Xcode
	- Vim 
	- Cocoapods
	- 爬坑

---

这里主要总结一下开发 iOS 使用工具上遇到的一些问题。

<!--more-->

## 如何为一个 workspace 中的多个 project 使用 cocoapods

一般一个 app 只有一个 project（也可以有多个，**其实 project 不是关键，它只是包含了一些配置信息，真正起作用的是 target，project 只是 target 的容器，有相互依赖关系的是 target**），在这个 project 中可以有多个 target 比如 `Myapp`、`Myapptest` 等，如下图：

![原理1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_xcode_1.png?raw=true)

这些 target 可以相互依赖，也可以依赖许多子 target，需要在 `Target Dependencied` 中注册：

![原理1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_xcode_2.png?raw=true)

那么如何为多个 project 中的各个 target 集成 pod 呢？就像上面说的 project 不重要，我们只要在设置target 的时候，引入 project 的路径即可：

```ruby
# 指定  workspace 
# 指定后不会生成默认的 workspace 文件
workspace 'TheApp.xcworkspace'

#  指定 xcodeproj
xcodeproj 'TheApp.xcodeproj'

# cocoapods 会用在 swift 项目
use_frameworks!

# 指定 target
target 'aProject' do
    # 由于 Podfile 和  .xcodeproj 不在同一文件夹下
    # 需要指定路径
    #  xcode 项目可以存在任何的子路路径中，只需要指定正确的路径就可以
    project ‘aProject/aProject.xcodeproj’

    #  在每一个 project 下都导入 SnapKit 进行测试
    pod "SnapKit"
end

target 'bProject' do
    project 'bProject/bProject.xcodeproj’
    pod "SnapKit"
end

target 'aFramework' do
    project ‘aFramework/aFramework.xcodeproj’
    pod "SnapKit"
end

target 'bFramework' do
    project ‘bFramework/bFramework.xcodeproj’
    pod "SnapKit"
end
```

可以看到，对于不同 project 中的 target，只要指定其与 `Podfile` 的相对位置就可以了。

假如 target `aProject` 下还有其他子 target，这些子 target 是可以直接引用父 target 引入的 SnapKit 组件的。

[CocoaPods 使用](http://www.jianshu.com/p/5a74c0842cf2)

[iOS 如何在一个存在多个project的workspace中引入cocoapods管理第三方类库](https://yq.aliyun.com/articles/8315)



## 安装 XVim

慕名下载了[XVim](https://github.com/XVimProject/XVim)，但是安装起来有点曲折。

> **现在统一使用 XVim2 了**

> 注意，创建自签名证书的时候要选对证书类型，要选为"代码签名"

![codesign](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/codesign.png?raw=true)

创建好自签名证书后，**要对 Xcode 进行重新签名**：

```
 $ sudo codesign -f -s zachary /Applications/Xcode.app
```

> 每次更新 Xcode 后，都要先对 Xcode 进行重新签名！

这个要等一段蛮长一段时间，这段时间可以先把代码码起来。

重新签名后，就可以在 clone 下来的 XVim 文件夹下编译了，需要 `cd XVim/`：

```
$ make
```

至于安装指南上的这个指令，其实不用太在意:

```
$ xcode-select -p
```

这个指令主要是看看 Xcode 的版本是否指定在 `/Applications/Xcode.app/Contents/Developer` 目录下，不是的话要手动指定。

重启xcode，会提示是否使用Xvim，选择`load`就OK了。

### 使用
除了Vim自身的命令外,XVim还有几个Xcode命令:

![XVim_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_1.png?raw=true)

附带Vim使用详解：
![XVim_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_2.png?raw=true)



## Podfile.lock
在协同开发时，经常会遇到，Podfile.lock 不一致，导致编译失败，需要重新 `pod install` 的情况。这里将简单介绍下 Podfile.lock
### Podfile.lock的作用
#### Podfile.lock
Podfile.lock 是在第一次运行 `pod install` 时候自动生成的。Podfile.lock 中会标注项目当前依赖库的准确版本，其中包括了项目在 Podfile 中直接标注使用的库，以及这些库依赖的其他库。这样的好处是当你跟小伙伴协同开发时，你的小伙伴同步了你的 Podfile.lock 文件后，他执行 `pod install` 会安装 Podfile.lock 指定版本的依赖库，这样就可以防止大家的依赖库不一致而造成问题。因此，CocoaPods 官方强烈推荐把 Podfile.lock 纳入版本控制之下。

但是，Podfile.lock 并不是一成不变的，当你修改了 Podfile 文件里使用的依赖库或者运行 `pod update` 命令时，就会生成新的 Podfile.lock 文件。所以，协同开发时需要注意使用 `pod install` 和 `pod update` 的区别:
- 使用 `pod install`，你只会安装 Podfile 中新改变的东西，并且会：优先遵循 Podfile 里指定的版本信息；其次遵循 Podfile.lock 里指定的版本信息来安装对应的依赖库。比如：下面在 Podfile 里没指定 iRate 的版本，但是 Podfile.lock 里指定了 iRate 的版本是 1.11.1，那么即使现在有最新的 1.11.4，最终也会安装 1.11.1。但是如果 Podfile 里指定了 iRate 版本是 1.11.3，那么则会安装 1.11.3，并更新 Podfile.lock 里的信息。
- 使用 `pod update`，你会根据 Podfile 的规则更新所有依赖库，不会理睬现有的 Podfile.lock，而是根据安装依赖库的情况生成新的 Podfile.lock 文件。

#### Manifest.lock
那么是如何知道拉取的 Podfile.lock 和 Pod 内，库的版本不一致的呢？在每次生成 Podfile.lock 的时候，都会在 Pod内生成一个 Podfile.lock 的副本Manifest.lock。由于 Pod 一般不会上传版本控制，Manifest.lock 就代表了本地的 Pod 版本。如果拉取了别人的 Podfile.lock，那么 Podfile.lock 和 Manifest.lock 就会产生不一致，就会导致编译失败。

#### 目的
有了这个检查机制就能保证开发团队的各个小伙伴能在运行项目前更新他们的依赖库，并保持这些依赖库的版本一致，从而防止在依赖库的版本不统一造成程序在一些不明显的地方编译失败或运行崩溃。

#### Podfile.lock 不同的可能原因
有时可能 Podfile 是相同的，但是你们的 Podfile.lock 还会不同。下面 `Podfile.lock 的引号问题`就是其中之一。

除了 gem，ruby等库的版本不同导致之外，还有可能是因为本地的 Podspec 文件不同导致的。所有的项目的 Podspec 文件都托管在 `https://github.com/CocoaPods/Specs`。第一次执行时，CocoaPods 会将这些 podspec 索引文件更新到本地的 `~/.cocoapods/` 目录下。如果长时间不更新 podspec，那么所要下载的直接依赖库或间接依赖库的最新版本可能发生了变化，就会导致安装了不一样的依赖版本，那么 Podfile.lock 的记录就不一样了。建议可以执行 `pod repo update` 更新一下 spec repo(一般情况下，执行 `pod install` 时会先自动更新 spec 的，除非向上面说的使用 `pod install --no-repo-update`，当然，现在只是记录下可能导致 Podfile.lock 不一致的原因，万一真的发生了呢？)



### Podfile.lock 的引号问题
最近在协作的时候遇到了这样的问题，我的 pod lock 总是和同事的不一致，差别在于是否有引号。这就导致了每次拉取别人的代码我都要 pod install 一遍，别人也是一样。
![podfile.lock](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/podlock.png?raw=true) 

查了一些资料[Inconsistent Podfile.lock files between Ruby versions](https://github.com/CocoaPods/CocoaPods/issues/3452),[Strange quotes in Podfile.lock?!](https://github.com/CocoaPods/CocoaPods/issues/6255) 发现大致是因为 gem 的一个 YAML 解析工具 psych 版本不同导致的。所以，解决问题的关键不是更新 cocoapods 而是更新 psych (更新 psych 前，先把 ruby，gem也先更新了，以防有什么差错).

![安装psych](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/安装psych.png?raw=true)



## 注释

### 方法和属性的快捷注释

在 Xcode8 之前，想要一键为属性和方法注释都需要通过插件完成。现在 Xcode8 中有了内置的注释方式： `alt+command+/`，只要将光标移动到想要注释的方法上，就会自动识别方法的参数和返回值，提供注释模板，非常方便。

### Xcode断点调试值都为nil的问题  

在Build Settings中 Optimization Level 设置成 None：

![debug](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/xcode调试.png?raw=true)

### #pragma mark

好的习惯从`#pragma mark`开始。就像这样：

```objc
@implementation ViewController

- (id)init {
  ...
}

#pragma mark - UIViewController

- (void)viewDidLoad {
  ...
}

#pragma mark - IBAction

- (IBAction)cancel:(id)sender {
  ...
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
  ...
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
  ...
}
```



在你的 `@implementation` 中使用 `#pragma mark` 来将代码分割成逻辑区块。这些逻辑区块不仅仅使得阅读代码本身容易许多，也为Xcode源导航增加了视觉线索（`#pragma mark` 声明前有一个水平分割并由破折号（`－`）开始）。

![pragma](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/pragma.png?raw=true)



## Xcode 并排查看代码

如果想要单纯分为左右两屏，按住 `alt` 再点击文件，即可在辅助窗口查看。

可以点击副主窗口右上角的 “+”，添加更多辅助窗口，此时按住 `alt` 就会让你选择在哪个辅助窗口查看文件代码。





