title: 使用 cocoapods 创建一个仓库
date: 2016/9/23 10:07:12  
categories: iOS
tags:
	- Cocoapods
---

想搞一个组件化的项目，所以先好好看下如何使用 cocoapods 创建一个私有库

<!--more-->

## Cocoapods 原理

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



