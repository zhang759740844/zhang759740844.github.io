title: iOS开发工具应用中的坑
date: 2016/9/23 10:07:12  
categories: iOS
tags:
	- Xcode
	- Vim 
	- Cocoapods
	
---

这里主要总结一下开发 iOS 使用工具上遇到的一些问题。

<!--more-->
## 安装 XVim

慕名下载了[Xcode](https://github.com/XVimProject/XVim)，但是安装起来有点曲折。

先用SourceTree创建了本地仓库，`clone`了代码。貌似Xcode8和Xcode7安装起来有点区别。

Xcode8 的用户要按照他的方法设置个人证书。主要是因为在 Xcode8 中，出于安全原因，官方已经不支持第三方插件了。想要使用第三方插件，就得开个后门，回到 Xcode7 的安全配置。操作没有歧义，按照教程就可以完成。

Xcode7不能用最新的版本，需要使用`commit`在`809527b`之前的版本。(MD，就不能加个tag!)需要自己创建一个本地branch。

之后有一步在下载下来的 XVim 文件夹下编译的操作，需要 `cd XVim/`：

```
$ make
```

至于下面这个指令，其实不用太在意:

```
$ xcode-select -p
```

这个指令主要是看看 Xcode 的版本是否指定在 `/Applications/Xcode.app/Contents/Developer` 目录下，不是的话要手动指定。具体这样是为了什么，我也没认真去搞明白。

重启xcode，会提示是否使用Xvim，选择`load`就OK了。

### 使用
除了Vim自身的命令外,XVim还有几个Xcode命令:

![XVim_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_1.png?raw=true)

附带Vim使用详解：
![XVim_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_2.png?raw=true)

## pod 的安装和使用
### 安装 cocoapods
安装 cocoapods 本身并没有难度。只是一点，Ruby 的默认源使用的是cocoapods.org，国内访问这个网址有时候会有问题，网上的一种解决方案是将远端替换成淘宝的。不过好像最近**淘宝的镜像也不能用了**，所以再换一个镜像吧：

```ruby
//删除默认源
$gem sources --remove https://rubygems.org/
//添加新源
$gem sources -a http://gems.ruby-china.org/
//查看
$ gem sources -l
```

这样再安装一次，就能安装成功了：

```
$ sudo gem install cocoapods
```

### pod install 卡在 Setting up CocoaPods master repo
安装完 pod 后就可以使用 `pod install` 安装第三方库了，但是第一次 `pod install` 的时候会自动从 `https://github.com/CocoaPods/Specs` 克隆到本地的 `.cocoapods` 目录下。

但是往往就会卡在这里，如果网络不畅，极有可能失败。因此，需要换手动克隆的方式。

首先进入该目录，如果没有，可以手动创建：

```
cd ~/.cocoapods/repos
```

克隆一个Specs库：
```
git clone https://github.com/CocoaPods/Specs
```

然后打开目录，我们可以看到这样的目录结构：

```
repos
	Specs
		CocoaPods-version.yml
		README.md
		Specs
		.git
```

现在将第一个 `Specs` 文件夹修改为 `master` 即可完成 `Setting up CocoaPods master repo` 的操作。

目录结构为：
```
repos
	master
		CocoaPods-version.yml
		README.md
		Specs
		.git
```

现在可以放心 `pod install` 了。

### pod install 卡在 Analyzing dependencies
原因在于当执行以上两个命令的时候会升级 CocoaPods 的 spec 仓库，加一个参数可以省略这一步，然后速度就会提升不少。加参数的命令如下：

```
pod install --verbose --no-repo-update
```

其中 `--verbose` 表示显示详情。后面的 `--no-repo-update` 才是正真起作用的。


