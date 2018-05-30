title: Mac 使用技巧
date: 2016/7/31 10:07:12  
categories: iOS
tags:
	- 持续更新
---

使用 Mac 用到的一些方法

<!--more-->



### 装机必备

#### 安装 homebrew

```shell
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### 安装 cocoapods

##### 修改 ruby 源

```shell
//删除默认源
$gem sources --remove https://rubygems.org/
//添加新源
$gem sources -a http://gems.ruby-china.org/
//查看
$ gem sources -l
```

##### 安装 cocoapods

```shell
$ sudo gem install cocoapods
```

#### 安装 node

```shell
brew install node
```

安装好node自然就安装好 npm 了。

#### 使用 iterm2

iterm2 最强大的功能就是可以分屏。还有就是可以选中即复制，以及按住 command 点击就可以直接跳转到文件夹或者网页。

##### 下载主题

官方的颜色默认是黑色的，可以去[Iterm2-color-schemes](http://iterm2colorschemes.com/)下载主题，下载 tar.gz 或者 zip 的压缩包，然后会下载非常多的配色。然后按照如下图操作配色：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iterm_1.png?raw=true)。

推荐使用 **Solarized Dark Higher Contrast**

##### 下载 zsh

zsh 可以提供一个很强的命令补全的功能，反正不太记得的命令或者参数，只要按 tab 就行了。下载也比较简单：

```bash
curl -L https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh | sh
```

`curl` 是利用URL语法在命令行方式下工作的开源文件传输工具。

另外，下载之后终端会变得比较好看。

#### 下载字体 Fira

字体 Fira 非常圆润，适合作为代码编辑器的字体，下载也非常简单。但是比较字体包有点大：

```shell
brew tap caskroom/fonts
brew cask install font-fira-code
```

`brew` 表示 homebrew

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

### shell 脚本 sudo 命令不输入密码

#### expect

expect 可以在需要密码的时候输入密码。安装 expect 非常简单：

```shell
brew install expect
```

```bash
#!/usr/bin/expect
set timeout 30
spawn bash "./Untitled-1.sh"
expect {
  Password: { send xxxx\r }
  Password: { send xxxx\r }
  Password: { send xxxx\r }
}
interact
```

`#!/usr/bin/expect` 表示到 `interact` 为止，都使用 expect 执行

`spawn` 该命令用于启动一个子进程，执行后续命令.

`expect` 从进程接受字符串，如果接受的字符串和期待的字符串不匹配，则一直阻塞，直到匹配上或者等待超时才继续往下执行。上面的 `Password：` 就是执行 `sudo` 命令是会出现的。 这里模拟了三个会出现 `Password:` 输入密码的场景。

`send` 向进程发送字符串，与手动输入内容等效，通常字符串需要以’\r’结尾。

`interact` 该命令将控制权交给控制台，之后就可以进行人工操作了。通常用于使用脚本进行自动化登录之后再手动执行某些命令。如果脚本中没有这一条语句，脚本执行完将自动退出。

#### sudo 不要密码

除了用 expect 在需要密码的时候输入外，还可以设置 sudo 不需要密码。虽然不安全，但是一般个人使用也不会有什么问题。

输入 `sudo visudo` 会打开 `/etc/sudoers` 文件，这就是要修改的配置文件。

然后再最后输入：

```
zachary ALL=(ALL) NOPASSWD: ALL
```

保存并推出。此时 sudo 就不会再需要密码了。

那么这句话是什么意思呢？

第一个字段 `zachary` 表示使用 sudo 命令的用户为 zachary

第二个字段，等号左边的 `ALL` 表示允许使用 sudo 的主机。这里是所有主机。

第三个字段，等号右边的 `(ALL)` 表示使用 sudo 后一什么身份来执行命令。

最后一个字段 `NOPASSWD: ALL` 表示所有命令，都不需要密码。

### 管道与 xargs

#### 管道

管道的作用就是把前一个命令的标准输出作为后一个命令的标准输入，比如：

```shell
ls -a | grep myfile
```

上面的命令就是把列出所有文件的输出，作为正则匹配查找的输入。

但是很多程序是不处理标准输入的，例如 kill , rm 这些程序如果命令行参数中没有指定要处理的内容则不会默认从标准输入中读取，所以：

```shell
ls -a | grep myfile | rm -f
```

这个命令是无法删除 myfile 相关的文件的。这时候就需要使用 xargs

#### xargs

xargs 和管道有什么不同可以看下面这个例子：

```shell
echo '--help' | cat 
输出：
--help

echo '--help' | xargs cat 
输出同 cat --help
```

可以发现，xargs 将其接受的字符串 `—help` 做成cat的一个命令参数来运行cat命令。

所以上面的删除方法要改成如下，就是可以达到目的的：

```shell
ls -a | grep myfile | xargs rm -f
```

xargs 还有许多参数，可以用在需要多个参数的情况，如果有需求，可以查阅使用。

### 使用 npmrc 登录 npm

我们要发布自己的 npm 包的时候，需要先登录到 npm。那么，npm 怎么验证发布的是本人呢？当我们在 shell 里登录到 npm 的时候，在 home 目录下会生成一个 `.npmrc` 文件。这个文件里有个 `authToken` ，同时，你的 npm 账户里的 token 列表里也会产生一个相应的 token。当你要发布代码的时候，npm 就会比较账户里的 token 是否包含这个 `authToken`。如果你手动删除了 npm 账户里的 token，那么之后也就无法通过该 `authToken` 登录了。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/auth_token.png?raw=true)

另外，每个项目里都可以有一个 `.npmrc` 使用不同的 token。项目里没有 token 的情况下才回去 home 下查找 `.npmrc`

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

### 安装的Mac应用显示已损坏无法打开

设置 => 安全性与隐私 => 通用 => 允许从以下位置下载的应用 => 任何来源

如果没有 任何来源 这个选项，需要执行命令：

```shell
sudo spctl --master-disable
```



### 如何显示/不显示隐藏文件

**在终端中执行如下代码并重启动Finder。**

```
显示隐藏文件：
defaults write com.apple.finder AppleShowAllFiles -bool TRUE
停止显示隐藏文件：
defaults write com.apple.finder AppleShowAllFiles -bool FALSE
```
**新系统中更好的方式：**

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