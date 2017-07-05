title: 使用 cocoapods 创建一个仓库
date: 2017/6/30 10:07:12  
categories: iOS
tags:
	- Cocoapods
---

想搞一个组件化的项目，所以先好好看下如何使用 cocoapods 创建一个私有库

<!--more-->

## Cocoapods 简单原理

之前的文章中我提到过，加速 `pod install` 的一个方法是 `pod install--no-repo-update `，下面就要解释一下这个后缀是要干什么。

### Cocoapods 本地目录

我们首先进入 Cocoapods 的本地目录：

```shell
$cd ~/.cocoapods/repos/master
```

可以发现这其实是一个 git 仓库：

```shell
$ git remote -v

origin	https://github.com/CocoaPods/Specs.git (fetch)
origin	https://github.com/CocoaPods/Specs.git (push)
```

进入 `Specs` 文件夹，会有很多歌0~f的文件夹嵌套，一层一层点下去，可以看到可以看到一些熟悉的第三方库，随便选一个点进去，是各个版本的文件夹，再随便找个文件夹进入，是一个json文件。

![原理1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_principle_1.png?raw=true)

打开这个json文件，里面记录了这个版本的库的一些信息，包括这个库的远程仓库的地址：

![原理2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_principle_2.png?raw=true)

所以我们就能大概明白了，每次我们 `pod install` 后，就会在这个 `Specs` 下查找对应的库的对应版本，然后找到相应的远程仓库，去远程仓库拉取代码。

### Pod install 过程

在 install 前，我们要编写一个 `Podfile`。整个依赖安装大概有四个部分：

- 解析 Podfile 中的依赖
- 下载依赖
- 创建 `Pods.xcodeproj` 工程
- 集成 workspace


Podfile 是 ruby 的语法，在 install 过程中，执行 Podfile 文件，将会获取到要依赖的库以及库的版本号。然后在本地的 `Specs` 中进行比对，找到对应的远程仓库的地址。

大部分的依赖都会被下载到 `~/Library/Caches/CocoaPods/Pods/Release/` 这个文件夹中，然后从这个这里复制到项目工程目录下的 `./Pods` 中。

如果 `Specs` 中如果没有找到 `Podfile` 中解析的库，就会报错。那么如果我们很久没有手动更新 `Spec` 了呢？不用担心，每次 `Pod install` 的时候，都会自动先去更新一下 `Specs`：`pod repo update`，当网络不好的时候就会很慢，所以才有了 `pod install --no-repo-update`，在 install 时，不更新 `Specs` 的命令。

`pod install` 完成后，会自动生成一个 `Podfile.lock`，用来锁定版本。我们不应该将其添加到 `.gitignore` 中去。只有在 `pod update` 后，才会自动更新 `Podfile.lock`。

CocoaPods 通过组件 CocoaPods-Downloader 已经成功将所有的依赖下载到了当前工程中，这里会将所有的依赖打包到 `Pods.xcodeproj` 中。主要做了几件事：生成 `Pods.xcodeproj` 工程；将依赖中的文件加入工程；将依赖中的 Library 加入工程；设置目标依赖（Target Dependencies）

这几件事情都离不开 CocoaPods 的另外一个组件 Xcodeproj，这是一个可以操作一个 Xcode 工程中的 Group 以及文件的组件，我们都知道对 Xcode 工程的修改大多数情况下都是对一个名叫 `project.pbxproj` 的文件进行修改，而 Xcodeproj 这个组件就是 CocoaPods 团队开发的用于操作这个文件的第三方库。

最后的这一部分与生成 `Pods.xcodeproj` 的过程有一些相似。首先会创建一个 workspace，之后会获取所有要集成的 Target 实例，然后将每一个 Target 加入到了工程，使用 Xcodeproj 修改 `Copy Resource Script Phrase` 等设置，保存 `project.pbxproj`，整个 Pod install 的过程就结束了。

[CocoaPods 都做了什么?](http://draveness.me/cocoapods.html)

[深入理解 CocoaPods](https://objccn.io/issue-6-4/)



### Podfile 中的一些关键词

#### :path

如果是我们自己开发的私有库，并且在开发阶段的情况下，可能就希望开发模式进行引用，则可以使用path参数：`:path => '~/Documents/AFNetworking'`

```ruby
target ‘Mike’ do
pod 'RNFS', :path => '../node_modules/react-native-fs'
pod 'React', :path => '../node_modules/react-native', :subspecs => [
  'Core',
  'RCTText',
  'RCTNetwork',
  'RCTWebSocket',
  'RCTImage',
  'RCTAnimation',
]
end
```

#### platform

这个参数是只依赖的库希望在哪个平台被编译。直接使用`platform :ios, '9.0'`。说希望采用iOS9.0的进行编译。

#### use_frameworks!

这个指明编译成动态库，而不是静态库，特别是在使用Swift库的过程中，特别需要使用这句。不过他会把所有项目的编译动态库，这一点有点不好。不过在使用Swift库的过程中就没办法了。

#### source

这个参数是指`Cocoapods`从哪些仓库(`Spec`)中获得框架的源代码，如果在结合使用开源库以及自己私有库的情况下，这个参数还是非常有意义的。在用到自己私有库的情况下只需要在`Podfile`文件开头列出你需要引用库的所有仓库地址即可。

```ruby
source 'https://github.com/artsy/Specs.git'  
source 'https://192.168.0.90:8888/MySepcs/Specs.git'  
```

最后一个官方 demo

```ruby
# open source
source 'https://github.com/CocoaPods/Specs.git'

# my work
source 'https://github.com/Artsy/Specs.git'

target 'App' do

  pod 'Artsy+UIColors'
  pod 'Artsy+UIButtons'

  pod 'FLKAutoLayout'
  pod 'ISO8601DateFormatter', '0.7'
  pod 'AFNetworking', '~> 2.0'

  target 'AppTests' do
    pod 'FBSnapshotTestCase'
    pod 'Quick'
    pod 'Nimble'
  end
end  
```

## 创建一个公有库

###  注册 CocoaPods 账号

首先要注册一个 CocoaPods 账号，在终端使用 `pod trunk` 命令注册，之后会有一封确认邮件，激活账号：

```shell
$ pod trunk register 759740844@qq.com 'Zachary' --verbose
```

激活成功后，再到终端输入，可以看到注册信息：

```shell
pod trunk me
```

```shell
ZacharydeMacBook-Pro:1.0.0 zachary$ pod trunk me
  - Name:     Zachary
  - Email:    759740844@qq.com
  - Since:    June 29th, 21:08
  - Pods:     None
  - Sessions:
    - June 29th, 21:08 - November 4th, 21:09. IP: 140.207.1.250
```



### 创建 .podspec

这一部分之前的文章[打包静态库](https://zhang759740844.github.io/2016/10/17/打包静态库/)里有较为详细的说明。为方便查看，复制了过来。

#### 创建工程

只需要输入 pod 的 `lib` 命令即可完成初始项目的搭建:

```ruby
pod lib create StaticWithCocoapods
```

输出指令后，会提示确认五个问题，按需求回答即可：

![创建库](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_2.png?raw=true)

稍等片刻，就会自动生成一个工程。

#### 配置信息

在项目目录下有一个 `xxx.podspec` 配置文件，需要进行修改，摘录如下：

```ruby
Pod::Spec.new do |s|
  s.name             = 'StaticWithCocoapods'
  s.version          = '0.1.0'
  s.summary          = 'A short description of StaticWithCocoapods.'

# This description is used to generate tags and improve search results.
#   * Think: What does it do? Why did you write it? What is the focus?
#   * Try to keep it short, snappy and to the point.
#   * Write the description between the DESC delimiters below.
#   * Finally, don't worry about the indent, CocoaPods strips it!

  s.description      = <<-DESC
TODO: Add long description of the pod here.
                       DESC

  s.homepage         = 'https://github.com/zhang759740844'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'Zachary' => '759740844@qq.com' }
  s.source           = { :git => '/Users/zachary/Desktop/StaticWithCocoapods', :tag => s.version.to_s }

  s.ios.deployment_target = '8.0'

  s.source_files = 'StaticWithCocoapods/Classes/**/*'
  
  s.resource_bundles = {
    'StaticWithCocoapods' => ['StaticWithCocoapods/Assets/*.png']
  }

  s.public_header_files = 'StaticWithCocoapods/Classes/**/*.h'
  s.frameworks = 'UIKit', 'MapKit'
  s.dependency 'AFNetworking', '~> 2.3'
  s.dependency 'SVProgressHUD'		
  s.subspec 'Aswift' do |a|
    a.source_files = 'A_swift/A_swift/**/*'
  end
end
```

- s.version 表示的是当前类库的版本号
- s.source 表示当前类库源
- s.sources_files 表示类库的源文件存放目录
- s.resource_bundles 表示资源文件存放目录
- s.frameworks 表示类库依赖的framework
- s.dependency 表示依赖的第三方类库，如果有多个要写多个 s.dependency
- s.subspec 表示依赖的子 podspec

其中要说明的是： 

1. source 可以填写远端 git 仓库，也可以是像我写的那样的本地 git 仓库。
2. 依赖项不仅要包含你自己类库的依赖，还要包括所有第三方类库的依赖，只有这样当你的类库打包成 .a 或 .framework 时才能让其他项目正常使用。
3. source_file 路径中出现的通配符 `*` 表示匹配任意字符， `**` 表示匹配所有当前文件夹和子文件夹。
4. source_bundles 中花括号内的 `'StaticWithCocoapods'` 就表示一个 `StaticWithCocoapods` bundle。
5. 这里的 dependency 一般情况下是指 cocoapods 的官方库。当然你也可以给自己创建的库添加自己的私有库，依赖同样是直接写在 dependency 里。由于 **dependency 中无法指定地址**。因此，如果添加私有库的时候要在使用这个库的工程的 Podfile 要添加你私有库的地址。这样的话，在下载 s.dependency 时，官方库找不到的情况下，就会到私有库中查找。
6. 有了 s.dependency 为什么还要用 s.subspec 呢？说明5中说到，dependency 无法指定地址。会有两种情况：1.别人用你的库的时候要在工程的 Podfile 文件中添加你的私有库的地址。2.如果你想用 cocoapods-packager 将你的库打包，你只能将你的私有库手动添加到项目的项目中。所以综上，你的私有库最好不要使用 s.dependency 的形式添加，使用 s.subspec 添加你的私有库信息，你的子私有库中的文件将直接回被添加到父库中编译。比如我在一个 `test_oc` 的库中添加了 `Aswift` 的 subspec，在 pod install 后得到如下文件结构：

![subspec](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_sub.png?raw=true)

#### 添加文件

向 **sources_files** 和 **public_header_files** 以及 **resource_bundle** 中添加图片和类文件。在 demo 的文件夹下执行 `pod install`。现在打开 demo 工程，可以看到创建的 `StaticWithCocoapods` 库的文件结构如下图：
![文件结构](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_1.png?raw=true)

> 现在工程里是带有 demo application 的，不过不用担心，在.podspec 中已经设置了源文件的目录，不会把 demo 中的各种测试文件也打包进去的。
>
> 不要在意上面图片上的图片路径 和 .podspec 中设置的 s.resource_bundles 路径不一致。这里的 Resources 看似是个文件夹，其实是一个 group，不是实际文件路径的地址。本例中实际的地址就是  `['StaticWithCocoapods/Assets/*.png']` 这个路径。

##### 加载图片资源的问题

到这里一切正常，也可以使用 `SVProgressHUD` ，但是当我想用 `[UIImage imageNamed:[[[NSBundle mainBundle] pathForResource:@"StaticWithCocoapods" ofType:@"bundle"] stringByAppendingString:@"/author.png"]]` 加载图片资源文件时，一直返回 `nil` 。最后在 Stackoverflow 的一个评论里总算找到了解决方法[[Cocoapods]:Resource Bundle not accessbile](http://stackoverflow.com/questions/25402782/cocoapodsresource-bundle-not-accessbile)

原先 demo 中 prdfile 的内容如下：

```ruby
use_frameworks!

target 'StaticWithCocoapods_Example' do
  pod 'StaticWithCocoapods', :path => '../'

  target 'StaticWithCocoapods_Tests' do
    inherit! :search_paths

    pod 'FBSnapshotTestCase'
  end
end
```

现在要删除 `use_frameworks!` 以及其相关内容，变成这样：

```ruby
target 'StaticWithCocoapods_Example' do
  pod 'StaticWithCocoapods', :path => ‘../‘
end
```

再次尝试加载图片，可以得到正确结果. ^_^。但是为什么会这样？

`use_frameworks!` 的添加与否，表示将 pod 中的库打包成静态库.a 还是动态库.framework。其实就是图片在盗宝成静态库或者动态库的时候所处的 bundle 不同。

如果打包成了 .a，那么编译的时候库里的所有文件其实是被打包在主工程下的，即 mainBundle 中，即可像通常那样在 mainBundle 中获取 StaticWithCocoapods.bundle 中的图片。

如果打包成了 .framework ，动态库里的文件不再被打包在主工程下了，而是在动态库自己的 bundle 中，比如本例中的动态库 bundle 路径如下：

![.a](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_bundle_1.png?raw=true)

此时就不能再在 mainBundle 中，而需要在动态库中找到 StaticWithCocoapods.bundle 下的文件了。通过的做法肯定不能直接获取 mainBundle，而是可以通过`[NSBundle bundleForClass:self.class]` 获取当前类所在的动态库的bundle，所以获取 `UIImage` 的通用方式为：

```objc
UIImage *image = [UIImage imageNamed:[[[NSBundle bundleForClass:self.class] pathForResource:@"StaticWithCocoapods" ofType:@"bundle" ] stringByAppendingString:@"/author.png"]];
```

##### 测试类库

在 demo application 中，如果把自己的库使用 pod 引入，注意要修改 Podfile 中库的路径为本地 `podspec` 所在的路径，不然每次都要从远端拉了。另外还有一个问题是这样每次更新库里的内容，运行的时候都要先 pod install 一下，这样非常的麻烦。

因此，我们可以直接将库里的文件添加到 demo application 中，同时要修改 Podfile，使其不要 pod 自身，而是 pod 其依赖的其他类库。这样做就是把类库的文件放到主工程下，就不用每次都 pod install 更新修改了。比如：

```ruby
# 原来可能是这样的
target 'StaticWithCocoapods_Example' do
  pod 'StaticWithCocoapods', :path => ‘../‘
end

# 现在要把依赖 StaticWithCocoapods 的类库加上，删除自己：
target 'StaticWithCocoapods_Example' do
  pod 'SVProgressHUD'
end
```

不用担心这样做不会产生任何问题。demo 中的文件依赖的改动并不影响 `.podspec` 中的设置，只要文件还是放在 `.podspec` 的指定路径下就行。发布类库的时候还是按照 `.podspec` 进行的，所以没有任何影响。

但是要注意一点，虽然你把文件放在主工程下编译，但是代码中获取资源文件的时候绝对不能偷懒使用 mainBundle 加载。因为别人可能将你发布的库打成动态库放到自身的工程中，这就会造成上面加载图片资源一样的问题。所以加载资源文件的时候一定要用上面说的通用的方法。

#### 提交代码

1. 使用 sourcetree 添加本地仓库
2. 提交上面的所有改动
3. 为改动添加 tag 为 `0.1.0`

设置好后如图：
![提交代码](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_3.png?raw=true)

这里要注意，tag 一定要打上，版本控制的时候就是以 tag 来辨别的。

#### 验证类库

开发完成静态类库之后，需要运行pod lib lint验证一下类库是否符合pod的要求。添加 `--allow-warnings` 忽略警告：
![验证类库](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_4.png?raw=true)

#### 打包库

如果你要发布到 cocoapds，那么这个操作是不必要的。但是如果你想直接获得一个动态或者静态库，那么可以手动打包这个库。

打包库需要一个 cocoapods 的插件 **cocoapods-packager**来完成类库的打包。

在终端执行以下命令，安装插件：

```ruby
sudo gem install cocoapods-packager
```

在目标文件夹内执行以下命令，完成打包：

```ruby
pod package StaticWithCocoapods.podspec --force
```

打包成 `.framework` ，也可以用 `--library` 打包成 `.a` 。不过 cocoapods 之前有过打包成 `.a` 但是没有暴露出头文件的bug，懒得试有没有修复了，所以还是都打成 `.framework` 吧。

![打包过程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/framework_cocoapods_5.png?raw=true)

现在在目标文件夹下就会多出一个 `StaticWithCocoapods-0.2.0` 目录，里面是打包好的 framework 。

### 发布.podspec

最后一步：发布。在仓库目录下执行：

```shell
pod trunk push xxxxx.podspec
```

上述命令将会更新本地的 repo 目录，然后将其提交到`Cocoapods`官方的仓库里去。

提交完成后，我们可以通过一下命令查询是否上传成功：

```shell
pod search xxx
```

如果是多人开发，希望将其他人也加入到项目中去的话，可以通过以下方式：

```shell
pod trunk add-owner '项目名' '邮箱'
```



### 更新删除等操作

当版本更新后，需要做的就是改变 `xxx.podspec` 中的版本号，然后重复上一步的步骤发布即可。

至于删除废弃等操作可以详见`pod trunk` 的 help中：

![help](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_public_.png?raw=true)

[CocoaPods公有仓库的创建]()

[Cocoapods系列教程(二)——开源主义接班人](http://www.pluto-y.com/cocoapods-contribute-for-open-source/)



## 创建一个私有库

### 创建版本库（repo）

之前说到，在`~/.cocoapods/repos/` 目录下有一个 master 文件夹，对应着 cocoapods 官方的一个版本库，保存着各个第三方库的索引。我们要创建一个私有库，就得先创建一个私有的版本库来存放这些私有库的索引。

我们先在远端创建一个空的仓库test，然后再终端执行：

```shell
$ pod repo add test https://git.oschina.net/baiyingqiu/test.git
```

这个仓库就被 clone 了下来，作为本地的版本目录了，可以进入`~/.cocoapods/repos/` 文件夹查看：

![版本库](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/cocoapods_private_1.png?raw=true)

### 创建.podspec

和上面公有的无二致。将本地代码推送到远程，并打上 tag。

> 需要明确，cocoapods 是通过 tag 来区分版本信息的，而不是 branch。也就是说，不论你的 tag 是打在 master 还是 develop 亦或任意一个其他分支，cocoapods 都能够正确获取该 tag 所处的 commit 的文件。
>
> 所以你的私有库其实不用再开一个新的 repo 去存储，可以直接在当前工程下开发。新开一个私有库的分支，专门开发私有库也是没有问题的。

### 将描述文件推送到版本库

将我们私有库的描述信息 push 到刚才的版本库中：

```shell
$ pod repo push test xxx.podspec  --verbose --allow-warnings
```

要加上 `--allow-warnings` 不然还是不能通过。

现在就可以通过 `pod search xxx` 来搜索了。



### 使用私有库

使用私有库的时候要在使用的工程的 `Podfile` 中加上私有库的版本库的路径，若有还使用了公有的pod库，需要把公有库地址也带上。例子：

```ruby
source ‘https://github.com/CocoaPods/Specs.git’
source ‘https://github.com/zhang759740844/test.git’

platform :ios, '8.0'

target ‘Mike’ do
  pod 'xxx'
end
```

现在就可以 `pod install` 了。



[CocoaPods私有仓库的创建](http://qiubaiying.top/2017/03/10/CocoaPods私有仓库的创建/)

[Cocoapods系列教程(三)——私有库管理和模块化管理](http://www.pluto-y.com/cocoapod-private-pods-and-module-manager/)

[使用Cocoapods创建私有podspec](http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/)推荐