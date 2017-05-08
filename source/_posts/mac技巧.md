title: Mac 使用技巧
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- 持续跟新
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