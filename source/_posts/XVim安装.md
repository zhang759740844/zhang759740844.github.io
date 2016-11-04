title: Xcode插件XVim安装与使用
date: 2016/9/23 10:07:12  
categories: iOS
tags:
	- Xcode
	- Vim 

---

XVim是Xcode中支持Vim的插件，尝试安装和使用

<!--more-->
## 安装

慕名下载了[Xcode](https://github.com/XVimProject/XVim)，但是安装起来有点曲折。

先用SourceTree创建了本地仓库，`clone`了代码。貌似Xcode8和Xcode7安装起来有点区别。由于我的版本还是7.3.1，就省略了Xcode8的步骤。

Xcode7不能用最新的版本，需要使用`commit`在`809527b`之前的版本。(MD，就不能加个tag!)需要自己创建一个本地branch。

打开xcode，编译XVim。

接下来这部最坑，由于没有Linux经验，文档上直接
```
$ make
```
但是没说路径，我也不知道在哪里执行，一直都是报没有`makefile`的错。而且文档上
```
$ xcode-select -p
```
也误导了我。

网上搜了半天，google到了(Baidu这个垃圾)，在`clone`下来的XVim目录中执行，进入目录果然有`MakeFile`，好的，直接`make`。

重启xcode，会提示是否使用Xvim，选择`load`就OK了。

## 使用
除了Vim自身的命令外,XVim还有几个Xcode命令:

![XVim_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_1.png?raw=true)

附带Vim使用详解：
![XVim_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/XVim_2.png?raw=true)