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

#### Cydia Substrate

Cydia 自动安装了 Cydia Substrate，包含三个模块：

1. **MobileHooker：用于替换系统和应用的方法**。提供 `MSHookMessageEx` 和 `MSHookFunction` hook OC 和 C 函数，
2. **MobileLoader：用于将第三方动态库加载到运行的目标应用里**（注入 Reaveal 就是通过它）。首先通过环境变量 `DYLD_INSERT_LIBRARIES` 把自己加载到目标应用里，然后查找 `/Library/Mobile Substrate/DynamicLibraries/` 目录下所有的 plist 文件，如果 plist 文件的配置信息符合当前的应用，则通过 `dlopen` 函数打开对应的 dylib 文件
3. **Safe mode：如果插件导致 SpringBoard 崩溃，将会让设备进入安全模式，禁用所有的三方插件**

#### 越狱必备

通过 Cydia 安装一下插件：

1. adv-cmds：提供指令 `ps -A` 获取全部进程
2. appsync：修改应用的文件会导致签名验证错误，该插件会绕过系统的签名验证



