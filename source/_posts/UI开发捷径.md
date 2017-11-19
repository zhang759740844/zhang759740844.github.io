title: 《iOS UI 开发捷径》读书笔记
date: 2016/7/31 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

读了 《iOS UI 开发捷径》，书上的东西比较简单，也比较零碎。所以做个大致的整理。

<!--more-->

### Interface Builder 概要

- Bundle 主要用来保存可执行代码以及保存资源文件。添加到工程里的资源文件会在编译时复制到 main bundle 中。如果没有，那么在 Copy Bundle Resource 中添加。
- 开发 SDK 的时候，资源文件会单独打包到一个 Bundle 中（静态库中无法添加资源文件）。Bundle 保存在 main bundle 下，所以集成 SDK 的时候要去找对应 Bundle 下的资源。

### 使用 Inferface Builder

- 右键 IB 中的控件拖到代码中自动生成 property 或者 action
- 如果要为 VC 添加一个 xib，需要将 xib 的 view 和 VC 的 view 连线。
- 可以在一个 xib 中创建多个 View，然后通过 `loadNibNamed` 方法获取 View 的数组
- 创建 UIVIewController 的时候，系统会判断是否有同名的 xib 文件。如果有，就直接加载。如果没有就需要调用 `initWithNibNamed` 方法指定 xib 文件名。
- xib 嵌套，不能直接拖个 View 设置其 class。你只能拖一个占位的 UIView，然后用代码将要嵌套的那个视图 addSubView 的方式添加进去。
- 加载 image 有两种方式，一种是通过 `initWithName ` 在 mainBundle 中加载；一种是通过 `initWithContentofFile` 通过路径拼接加载。前者加载的图片会被系统缓存，后者则不会。所以大图片最好用后者。
- 要创建 Bundle 要在 macOS 中找，然后将 Supported Platforms 设置为 iOS
- 只能向 sb 文件中添加 UIViewController，而不能添加 View。
- 一个 sb 可以添加多个 VC，通过 storyboardID 来识别sx