title: Git技巧
date: 2016/8/27 14:07:12  
categories: Git
tags: [Git]

---

虽然一直使用sourcetree，还是应该对git指令做一些了解的。本篇将列举使用中遇到的关于git的一些使用技巧。不定期更新。

<!--more-->

### git rm
ios中pod的第三方库经常会被`gitignore`掉，让使用者自己下载。

想要达到一个效果：使文件脱离git的版本管理，但不是会删除它.



#### 使用命令行
可以使用`git rm --cached file`
Git 将不再跟踪此文件，尽管它仍然是在您的硬盘上.
对于文件夹，可以使用`git rm -r --cached file`取消关联整个文件夹。

#### 使用sourcetree
在sourcetree中操作要麻烦一些。在添加完`gitignore`后，将想要ignore的文件移出git目录，然后`commit`该操作，再将文件移回来。这个时候文件就被ignore了。

需要注意的是，一定要先执行`commit`操作。因为，在`.git`中存在了pod的版本控制的文件关联，添加了`gitignore`后，git并不会主动将pod文件关联删掉。
因此，需要先用`commit`将该文件关联删除，才能正确ignore该文件夹。

#### 总结
这里的移除相当于执行了 `rm 文件`的操作，因而需要使用`commit`,而使用命令行的`git rm 文件`则是通过git指令，直接移除了关联。

### git add
不管是什么项目总是需要添加新文件的。添加新文件即add，但是从github上clone的框架总是不能自动添加到本地的git版本里。可能是由于clone的框架本生带有`.git`的缘故吧。

因此，如果当git不能自动添加文件关联的时候，需要使用`git add 文件`的方式手动添加。

使用sourcetree时，也可以将要添加的部分先保存在其他地方，然后将文件夹内的文件复制到想要放的地方。这样隐藏的`.git`不会被复制，文件也就能添加进sourcetree了。

### merge
merge可以说是很常用的一个功能，用于代码合并。**只要是关乎于不同branch的代码进行合并都可以用merge啊**。

#### 合并分支
merge 有一个名词叫“快进式合并”，合并的时候会直接将会直接将一个分支分支指向另一个分支：
![fast-merge](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fast-merge.png?raw=true)
但是有的时候，为了保证版本演进的清晰，我们希望在合并的分支上生成一个新节点表示做过合并操作，就要采用“非快进式合并”：
![no-fast-merge](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/not-fast-merge.png?raw=true)
那么在 sourcetree 怎么做呢？勾选“无论是否可以快进更新都创建新的提交”
![select](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/not-fast-merge2.png?raw=true)
最后提交合并的记录即可：
![commit](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/not-fast-merge3.png?raw=true)

#### 临时性分支
除了Master和Develop。前者用于正式发布，后者用于日常开发。其实，常设分支只需要这两条就够了，不需要其他了。

但是，除了常设分支以外，还有一些临时性分支，用于应对一些特定目的的版本开发。临时性分支主要有三种：
- 功能（feature）分支
- 预发布（release）分支
- 修补bug（fixbug）分支

这三种分支都属于临时性需要，使用完以后，应该删除，使得代码库的常设分支始终只有Master和Develop。

功能分支，它是为了开发某种特定功能，从Develop分支上面分出来的。开发完成后，要再并入Develop。

预发布分支，它是指发布正式版本之前（即合并到Master分支之前），我们可能需要有一个预发布的版本进行测试。预发布分支是从Develop分支上面分出来的，预发布结束以后，必须合并进Develop和Master分支。

软件正式发布以后，难免会出现bug。这时就需要创建一个分支，进行bug修补。修补bug分支是从Master分支上面分出来的。修补结束以后，再合并进Master和Develop分支。

几种分支的命名方式分别为 `feature-*`,`release-*`,`fixbug-*` 的形式。

### rebase
变基操作是git常用的一个功能，这里不再说明其作用，而是强调一下他的风险。

变基操作的一条准则是：**不要对在你的仓库外有副本的分支执行变基**。

> 变基操作的实质是丢弃一些现有的提交，然后相应地新建一些内容一样但实际上不同的提交。

也就是说：变基操作将强行改变仓库的提交情况。注意，是丢弃不是删除，git对于这种丢弃没有任何记录。

如下图所示，`C5`是其他分支的合并。先将master重置到`C1`的提交，然后rebase`C5`所在分支，得到新的提交`C4'`。一般情况下，git是不允许这样的提交到remote的，但是可以使用`git push --force`强制覆盖remote的仓库。这样`C4``C6`就不复存在了。
![git_rebase_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/git_rebase_1.png?raw=true)

话分两头，别人也有自己的本地master，并且别人已经合并了之前的提交`C6`到`C7`，如下图所示：
![git_rebase_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/git_rebase_2.png?raw=true)

由于别人的本地git并不知道`C4``C6`已经由于rebase被删除。比对仓库信息后，在它看来，就是remote的master有了一个新的提交`C4'`，于是就拉取，但是本地git一看远端提交`C4'`和本地提交`C4``C6`都修改了相同的文件，那就产生冲突了啊。那么别人就必须要合并这两条提交历史的内容，生成一个新的合并提交，最终仓库会如图所示：
![git_rebase_3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/git_rebase_3.png?raw=true)

这样就比较尴尬了，因为当时使用rebase就是为了删掉`C4``C6`，但是别人这么一合并一提交，`C4``C6`又全回来了。

如果只是单人开发，变基操作没有任何危险，想怎么变怎么变。但是在多人开发，切记**只把变基命令当作是在推送前清理提交使之整洁的工具，并且只在从未推送至共用仓库的提交上执行变基命令**。假如你在那些已经被推送至共用仓库的提交上执行变基命令，并因此丢弃了一些别人的开发所基于的提交，那么问题就大了。

### sourcetree 无法看到提交记录

点击 sourcetree 中提交记录，可以看到详细的提交信息。但是有些时候点击提交项的时候看不到详细的提交信息。

可以在终端内输入以下命令，然后重启 sourcetree 解决：

```
defaults delete com.torusknot.SourceTreeNotMAS "NSSplitView Subview Frames repowindow_LogViewDescSplitter"
```


### commit message 规范

#### 基本规范

第一行为对改动的简要总结，建议长度不超过 50 个字符，用语采用命令式而非过去式。第一行结尾不要留句号。第一个单词的第一个字母要大写。

第二行为空行。

第三行开始，是对改动的详细介绍，可以是多行内容，建议每行长度不超过 72 个字符。**主要用来写明白为什么要这样改**

避免注释不清晰。比如只有「修正 BUG」、「改错」、「升级」、「添加某文件」等，没有其他内容，等于没说。 6.「提交」的概念是具有独立的功能、修正等作用。小可以小到只修改一行，大可以到改动很多文件，但划分的标准不变，一个提交就是解决一个问题的。

#### 主要前缀

- Add 新加入的需求
- Fix  修正bug
- Change 修改功能
- Update 完成任务或者模块后的更新
- Remove 移除功能
- Refactor 重构功能

