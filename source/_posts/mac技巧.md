title: Mac 使用技巧
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- 持续更新
---

使用 Mac 用到的一些方法

<!--more-->

### VSCode 同步方案

VSCode 的插件 Setting Sync 提供了通过 github 的 Gist 完成配置同步的功能。但是由于它的教程不完整，导致同步起来会产生省问题。最常见的问题是无法下载配置，提示信息为：`Sync : Invalid / Expired GitHub Token. Please generate new token with scopes mentioned in readme. Exception Logged in Console.`

Gist 可以保存上传的配置文件。拉取配置文件需要配置两个 id，一个是 Gist Id，一个是 Token Id。这两个 Id 前者标识配置文件，后者用于身份验证。我们无法下载的原因就是我们使用单单在 `Sync:Download Settings` 命令中使用了 Gist id，所以错误提示才是无效的 token。

那么该怎么进行身份验证呢？还是在 VSCode 中输入命令：`Sync:Advanced Options`，然后选择 `Sync:Edit Extension Local Settings`，编辑 `syncLocalSettings.json` 这个配置文件。这个文件中有一项 token 没有设置，这里就需要设置为 Token Id。你可以用之前上传配置文件时设置的 Token，也可以再新建一个 Token。

### sudo 仍然无法获得权限

有些时候使用 sudo 命令仍然会失败，提示 *Operation not permitted*。这是因为开启了 sip 的缘故。我们需要关闭 sip：

1. 重启OSX系统，然后按住Command+R，进入**恢复模式**
2. 出现界面之后，选择屏幕左上角菜单栏中的**实用工具**中的**Terminal**，打开终端
3. 在Terminal中输入`csrutil disable` 关闭SIP
4. 重启reboot OSX

### 自制一个搜索 Markdown 文件的 workflow

可以自制一个 workflow 而不是使用 `open` 打开 markdown 文件。

主要步骤就是新建一个 workflow，然后 **右键→Inputs→File Filter**。这里的重点就是选择 File Filter，这样系统会自动进行文件搜索。然后在弹出的设置界面设置关键字，以及搜索的文件类型。你也可以在第二个选项卡中设置搜索路径：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/workflow1.png?raw=true)

然后设置这个操作的动作：**拉线→Actions→Open File**。Open File 出现的设置界面不用任何操作，直接点完成即可：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/workflow2.png?raw=true)

一般**我们使用的打开 Xcode 工程也可以这样设置。**

### Alfred3 开机访问通讯录

破解的 Alfred3 每次开机都要访问通讯录，解决办法是在终端中进行签名:

```shell
sudo codesign -f -d -s - /Applications/Alfred\ 3.app/Contents/Frameworks/Alfred\ Framework.framework/Versions/A/Alfred\ Framework
```



### 使用 iterm2

iterm2 最强大的功能就是可以分屏。还有就是可以选中即复制，以及按住 command 点击就可以直接跳转到文件夹或者网页。

#### 下载主题

官方的颜色默认是黑色的，可以去[Iterm2-color-schemes](http://iterm2colorschemes.com/)下载主题。按照上面提供的步骤设置。推荐使用 **Solarized Dark Higher Contrast**

#### 下载 zsh

zsh 可以提供一个很强的命令补全的功能，反正不太记得的命令或者参数，只要按 tab 就行了。下载也比较简单：

```bash
curl -L https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh | sh
```

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

> 不确定安装了 oh my zsh 会不会能直接改变字体颜色。如果不能就照着这个设置。



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
**更好的方式：**

可以使用快捷键 **shift+command+.**

### handoff 无法使用

handoff 是一个能在电脑和手机间共享信息的东西，但是有的时候就是识别不到。如果已经明确自己开启了 handoff 但是任然无法使用，那么可以先关闭电脑的蓝牙功能，然后再打开。这样就可以使用了。



### 如何破解欧路的使用次数限制

非注册用户只能使用 Xcode 50次，为了省这 98块钱就在网上找破解版，无意中搜到了修改使用次数的方法。

在 `~/library/preferences/` 文件夹里，找到 `com.eusoft.eudic.plist` 文件。用 Xcode 打开它，然后找到并修改 `MAIN_TimeLeft` 的值：

```xml
<key>MAIN_TimeLeft</key>	<integer>50</integer>
```

![修改](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oulupojie.png?raw=true)