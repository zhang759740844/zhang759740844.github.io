title: Git技巧
date: 2016/8/27 14:07:12  
categories: Git
tags: [Git]

---

虽然一直使用sourcetree，还是应该对git指令做一些了解的。本篇将列举使用中遇到的关于git的一些使用技巧。不定期更新。

<!--more-->

## git rm
ios中pod的第三方库经常会被`gitignore`掉，让使用者自己下载。

想要达到一个效果：使文件脱离git的版本管理，但不是会删除它.



### 使用命令行
可以使用`git rm --cached file`
Git 将不再跟踪此文件，尽管它仍然是在您的硬盘上.
对于文件夹，可以使用`git rm -r --cached file`取消关联整个文件夹。

### 使用sourcetree
在sourcetree中操作要麻烦一些。在添加完`gitignore`后，将想要ignore的文件移出git目录，然后`commit`该操作，再将文件移回来。这个时候文件就被ignore了。

需要注意的是，一定要先执行`commit`操作。因为，在`.git`中存在了pod的版本控制的文件关联，添加了`gitignore`后，git并不会主动将pod文件关联删掉。
因此，需要先用`commit`将该文件关联删除，才能正确ignore该文件夹。

### 总结
这里的移除相当于执行了 `rm 文件`的操作，因而需要使用`commit`,而使用命令行的`git rm 文件`则是通过git指令，直接移除了关联。

## git add
不管是什么项目总是需要添加新文件的。添加新文件即add，但是从github上clone的框架总是不能自动添加到本地的git版本里。可能是由于clone的框架本生带有`.git`的缘故吧。

因此，如果当git不能自动添加文件关联的时候，需要使用`git add 文件`的方式手动添加。

使用sourcetree时，也可以将要添加的部分先保存在其他地方，然后将文件夹内的文件复制到想要放的地方。这样隐藏的`.git`不会被复制，文件也就能添加进sourcetree了。

## merge
merge可以说是很常用的一个功能，用于代码合并。

不过由于之前没真正用过，用起来还比较死板。比如，学的时候留下刻板印象就是，将feature branch合并进develop用merge，但是刚刚遇到要把develop合并到feature branch的时候就忘掉merge这个功能了。**只要是关乎于不同branch的代码进行合并都可以用merge啊**。

好吧。可能只是我蠢而已。稍微做个记录，证明自己曾蠢过。但愿不要一直蠢下去。


## rebase
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