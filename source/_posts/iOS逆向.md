title: 《iOS 应用逆向与安全》笔记
date: 2018/10/31 14:07:12  
categories: iOS
tags: 

 - 学习笔记
	
---

很早就买了《iOS应用逆向与安全》这本书，现在把学到的内容总结一下。

<!--more-->

## 越狱设备

### Cydia

添加 *雷锋源（apt.abcdia.com）*



### SSH

cydia 中搜索并安装 dropbear 提供 SSH 功能。连接，默认密码为 *alpine*

```bash
ssh root@192.168.2.202
```

#### 修改密码

输入如下修改密码：

```bash
passwd
```

#### 公钥登录

在目标设备的 $HOME/.ssh 目录下找打 *authorized_keys* 文件，如果没有则自己创建。将本机的公钥复制到该文件中。

### iOS系统结构

#### 文件目录

要访问越狱设备的文件系统，要通过 Cydia 安装 *Apple File Conduit 2*.

Mac 上安装 *iFunBox*

#### 文件权限

通过 `ll` 可以查看文件的权限，一般权限包括三类：

1. 所有者权限：文件所有者进行的操作
2. 组权限：属于该组的成员对他能进行的操作
3. 其他人权限：其他人能进行的操作

