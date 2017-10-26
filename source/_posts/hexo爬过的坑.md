title: hexo 配置遇到的一些坑
date: 2016/12/3 14:07:12  
categories: Git
tags: 
	- Hexo
	- 爬坑

---

配置个人博客有时候是一件很蛋疼的事情，经常会出现一些奇奇怪怪的问题。这里我就将把我填上的一些坑总结一下。

<!--more-->

### 更新 hexo

自从把自己的博客构建起来后就没有升级过 hexo，这次要设置 apple-touch-icon，更新了 hexo，也花了小半天时间。记述一下，以后更新或者换电脑迁移就容易一些了。

#### 下载 npm

npm 是由 node 提供的。因此，到 node 的官网下载安装 node 即可获得 npm

#### 安装 hexo

使用如下命令安装 hexo 的最新版本

```shell
npm install hexo-cli -g
```

#### 初始化 hexo

如果你是更新 hexo，可能以为这步没有必要，这么想就错了。hexo 更新的时候会更新一些 node-modules，这些 node-modules 并不在 package.json 中记录，这些 module 是很重要的。所以不管是更新还是新建，你都必须要初始化 hexo。你可以在一个空文件夹中初始化好 hexo，然后把这些 module 拷贝到你当前的 node-module 文件夹内

```shell
hexo init
```

#### 下载 node-module

这步就不多说了

```shell
npm install
```

#### 下载 deploy module

hexo3 开始需要借助一个 deploy module 来推送博客，但这并没有在默认的 package.json 中，要自己动手添加：

```shell
npm install hexo-deployer-git --save
```

### 更新 next

next 肯定要和 hexo 同步更新呀，现在 next 主题非常人性化了，现在可以添加 apple-touch-icon，而且也有了默认的本地搜索。所以记得要仔细看配置文件，不要直接把以前的配置文件直接拷贝过来。

#### 更新 next

更新 next 的话，你可以直接把原来的 next 文件夹删掉，然后使用终端拉取：

```shell
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

#### 更改图片

包括 favicon，apple-touch-icon，avatar 都可以在 next 主题中随意替换。路径在 next 主题的 `_config.yml` 中配置，默认路径在 `themes/next/source/images` 内。

#### 搜索

在 `_config.yml` 中已经有了搜索的配置，你只要设置 enable 为 true 即可。然后就是下载搜索的 module，执行：

```shell
npm install hexo-generator-search --save
npm install hexo-generator-searchdb --save
```

### 各种部署的内容和本地不一致
有时候 hexo 会抽风，有些文件无论怎么 `hexo g` 都 deploy 不上去。这个时候可以输入 `hexo clean`，删除 database 和 public 文件夹，再 generate 和 deploy 就可以部署成功了。 

### 搜索功能打不开

写了一片文章后点击搜索无响应了。

这回的原因是在文章中的某一处多了一个空白字符，就是那种什么位置也不占，但是按一次 delete 才能删除的那种东西。把这个空白字符删掉就可以正常搜索了。

产生这种字符的原因不太清楚，排查起来也很麻烦。本来都是每次修改后，都要 `hexo g` 再 `hexo s` 的，很费力。不过，后来我发现如果启用本地服务器，即 `hexo s`，那么直接修改后，刷新 localhost 的网页(也可以不刷新)，就可以直接查看修改是否有效了。所以说本地服务器的搜索应该不走 `search.xml` ，而是直接拿取本地文件。这样调试起来方便许多。

排查的方法就是先删部分文件，看是否可以打开搜索功能，这样就能定位到某个文件，然后同样的方法定位到某一段，最后在这一段中不停的移动光标，如果有空白字符，那么会在某个地方移动光标的时候，光标不动。这个时候删除这个空白字符就行了。