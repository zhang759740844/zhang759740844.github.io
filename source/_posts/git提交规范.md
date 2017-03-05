title: Commit message 的格式
date: 2017/3/4 10:07:12  
categories: Git
tags:

	- Git

---

规范化提交信息的方式。

<!--more-->

## sourcetree 无法看到提交记录

点击 sourcetree 中提交记录，可以看到详细的提交信息。但是有些时候点击提交项的时候看不到详细的提交信息。

可以在终端内输入以下命令，然后重启 sourcetree 解决：

```
defaults delete com.torusknot.SourceTreeNotMAS "NSSplitView Subview Frames repowindow_LogViewDescSplitter"
```

## commit message 规范

### 基本规范

第一行为对改动的简要总结，建议长度不超过 50 个字符，用语采用命令式而非过去式。第一行结尾不要留句号。第一个单词的第一个字母要大写。

第二行为空行。

第三行开始，是对改动的详细介绍，可以是多行内容，建议每行长度不超过 72 个字符。**主要用来写明白为什么要这样改**

避免注释不清晰。比如只有「修正 BUG」、「改错」、「升级」、「添加某文件」等，没有其他内容，等于没说。 6.「提交」的概念是具有独立的功能、修正等作用。小可以小到只修改一行，大可以到改动很多文件，但划分的标准不变，一个提交就是解决一个问题的。

### 主要前缀

- Add 新加入的需求
- Fix  修正bug
- Change 修改功能
- Update 完成任务或者模块后的更新
- Remove 移除功能
- Refactor 重构功能