title: Mac 使用技巧
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- 持续更新
---

使用 Mac 用到的一些方法

<!--more-->

### 使用 iterm2

#### 下载主题

官方的颜色默认是黑色的，可以去[Iterm2-color-schemes](http://iterm2colorschemes.com/)下载主题。按照上面提供的步骤设置。推荐使用 **Solarized Dark Higher Contrast**

#### 设置终端字体颜色

下载好主题后，终端颜色并没有很好看，需要进一步设置，在终端输入 `vim ~/.bash_profile`

```shell
#enables colorin the terminal bash shell export
export CLICOLOR=1

#setsup thecolor scheme for list export
export LSCOLORS=gxfxcxdxbxegedabagacad

#sets up theprompt color (currently a green similar to linux terminal)
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;36m\]\w\[\033[00m\]\$ '
#enables colorfor iTerm
export TERM=xterm-256color
```

终端颜色就能非常漂亮了

#### 设置 Vim 颜色

在终端输入 `vim .vimrc`

```shell
 syntax on
 set number
 set ruler
```

这样 Vim 就能高亮提示了。



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