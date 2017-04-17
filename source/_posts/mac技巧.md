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