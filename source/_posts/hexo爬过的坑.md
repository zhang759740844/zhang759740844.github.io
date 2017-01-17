title: hexo 配置遇到的一些坑
date: 2016/12/3 14:07:12  
categories: Git
tags: 
	- Hexo
	- 爬坑

---

配置个人博客有时候是一件很蛋疼的事情，经常会出现一些奇奇怪怪的问题。这里我就将把我填上的一些坑总结一下。

<!--more-->

### 各种部署的内容和本地不一致
有时候 hexo 会抽风，有些文件无论怎么 `hexo g` 都 deploy 不上去。这个时候可以输入 `hexo clean`，删除 database 和 public 文件夹，再 generate 和 deploy 就可以部署成功了。 

### next主题空白
这是16年11月份的问题，突然打开自己的博客发现一片空白，但是在本地 `hexo s` 就能够显示出来。原本以为是我远端仓库出了问题，就重建了个新的，但是没有效果。很急很难受。网上搜索真的难，以为我不知道有没有人遇到过这样的问题，并且我也不知道该怎么去描述这个出现的问题。不过，几经波折，还是找到了解决的方案。

因为 Github 在 11月3日的更新中忽略了 `venders` 和 `node_modules` 两个文件夹。于是就打不开 `venders` 目录下的 js 文件了。

那么如何解决呢？`next` 的作者推荐我们修改 `venders` 文件夹名。比如将 `source/venders` 修改成 `source/libs`。同时，修改主题下的配置文件 `_config.yml`,将 `_internal: venders` 修改成之前修改的名字，如 `_internal: libs`。

当然，如果是直接克隆的 `next` 主题，可以通过 `git pull` 直接拉取 `next` 作者的修改。

### 缺少 DTraceProviderBindings 模块
这是在换了 Mac，重新安装 hexo 后，`hexo g` 时出现的问题。会出现如下 Error 提示：

```
{ [Error: Cannot find module './build/Release/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
{ [Error: Cannot find module './build/default/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
{ [Error: Cannot find module './build/Debug/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
```

虽然结果上来说并不影响 hexo 的执行，但是每次都提示还是很让人不爽的。于是，搜索了一下，网上确实有很多讨论这个问题的解决方法的。一般比较多的方式是使用 `npm install hexo --no-optional` 安装 hexo（一般安装还是将 hexo 安装成全局的吧，使用 `-g` 选项）。

但是我用了这个方式后，并没有效果，然后又查了几种方式都不行，就很气。然后就看 hexo 的安装文档，突然灵光一闪。因为现在安装 hexo 的方式变成了 `npm install hexo-cli -g`，那么是不是说原本的  `npm install hexo` 方式已经被废弃了呢？ 于是我就尝试在终端输入 `npm install hexo-cli -g --no-optional`。果然就没有这个 Error 了。

