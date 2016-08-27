title: Git技巧
date: 2016/8/27 14:07:12  
categories: Git
tags: [Git]

---

虽然一直使用sourcetree，还是应该对git指令做一些了解的。本篇将列举使用中遇到的关于git的一些使用技巧。

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