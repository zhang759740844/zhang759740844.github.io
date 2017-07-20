title: Mac 使用技巧
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- 持续更新
---

使用 Mac 用到的一些方法

<!--more-->

### 如何显示/不显示隐藏文件

**在终端中执行如下代码并重启动Finder。**

```
显示隐藏文件：
defaults write com.apple.finder AppleShowAllFiles -bool TRUE
停止显示隐藏文件：
defaults write com.apple.finder AppleShowAllFiles -bool FALSE
```
### handoff 无法使用

handoff 是一个能在电脑和手机间共享信息的东西，但是有的时候就是识别不到。如果已经明确自己开启了 handoff 但是任然无法使用，那么可以先关闭电脑的蓝牙功能，然后再打开。这样就可以使用了。



### 如何破解欧路的使用次数限制

非注册用户只能使用 Xcode 50次，为了省这 98块钱就在网上找破解版，无意中搜到了修改使用次数的方法。

在 `user/library/preferences/` 文件夹里，找到 `com.eusoft.eudic.plist` 文件。用 Xcode 打开它，然后找到并修改 `MAIN_TimeLeft` 的值：

```xml
<key>MAIN_TimeLeft</key>	<integer>50</integer>
```

![修改](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oulupojie.png?raw=true)