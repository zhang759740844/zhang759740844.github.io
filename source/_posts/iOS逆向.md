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

在目标设备的 $HOME/.ssh 目录下找 *authorized_keys* 文件，如果没有则自己创建。将本机的公钥复制到该文件中。

### iOS系统结构

#### 文件目录

要访问越狱设备的文件系统，要通过 Cydia 安装 *Apple File Conduit 2*.

Mac 上安装 *iFunBox*

#### 文件权限

通过 `ll` 可以查看文件的权限，一般权限包括三类：

1. 所有者权限：文件所有者进行的操作
2. 组权限：属于该组的成员对他能进行的操作
3. 其他人权限：其他人能进行的操作

可以通过 chmod 改变权限

#### Cydia Substrate

Cydia 自动安装了 Cydia Substrate，包含三个模块：

1. **MobileHooker：用于替换系统和应用的方法**。提供 `MSHookMessageEx` 和 `MSHookFunction` hook OC 和 C 函数，
2. **MobileLoader：用于将第三方动态库加载到运行的目标应用里**（注入 Reaveal 就是通过它）。首先通过环境变量 `DYLD_INSERT_LIBRARIES` 把自己加载到目标应用里，然后查找 `/Library/Mobile Substrate/DynamicLibraries/` 目录下所有的 plist 文件，如果 plist 文件的配置信息符合当前的应用，则通过 `dlopen` 函数打开对应的 dylib 文件
3. **Safe mode：如果插件导致 SpringBoard 崩溃，将会让设备进入安全模式，禁用所有的三方插件**

#### 越狱必备

通过 Cydia 安装一下插件：

1. adv-cmds：提供指令 `ps -A` 获取全部进程的进程ID和**可执行文件路径**（dumpdecrypted 砸壳时候用到）
2. appsync：修改应用的文件会导致签名验证错误，该插件会绕过系统的签名验证

> `ps -A`  获取完整的可执行文件路径

## 逆向工具详解

### 应用解密

#### dumpdecrypted

dumpdecrypted 会注入可执行文件，然后动态地从内存总 dump 出解密后的内容

1. github上下载，包含一个 makefile 和一个 .c 文件
2. `$make` 编译生成一个 **dumpdecrypted.dylib** 动态库
3. 远程登录到手机，将生成的动态库放到 `/var/root` 下
4. 在 `/var/root` 目录下使用环境变量 `DYLD_INSERT_LIBRARIES` 将 dylib 注入到要脱壳的可执行文件中
  1. `ps -A` 拿到正在运行的要注入的应用的完整路径（/var/mobile/Containers/…/{应用的名字}）
  2. 终端输入 `$DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib {完整路径}` 进行注入
5. 最终在 `/var/root` 目录下，得到的 `{应用名}.decrypted` 就是脱壳后的 mach-o。

#### Clutch

Cluch 会生成一个新的进程，然后暂停进程，dump 内存

1. github上下载源码
2. 使用 Xcode 编译，会在 *build* 目录下生成一个二进制文件 *Clutch*
3. 远程登录到手机，将 Clutch 放到 `/usr/bin` 目录下，并为其添加执行权限
4. `Clutch -i` 列出所有可以脱壳的应用，以及其 **BundleId**
5. `Clutch -b {BundleId}` 砸壳
6. 砸完壳的 ipa 的路径会在屏幕上输出

Clutch 砸壳得到的是一个包含各种资源文件的完整的 .app

> `Clutch  -i` 获取所有 BundleId

### class-dump

clas-dump 可以导出已砸壳应用的头文件。（未砸壳应用被加密无法获取，需要先砸壳）

官网下载 class-dump，把下载的二进制文件放到 `/usr/bin` 目录下。
```shell
$class-dump -H {二级制文件} -o {文件路径}
```

> 如果项目中包含 Swift 编写的代码，那么很有可能产生 `class-dump[13914:58557859] Error: Cannot find offset for address 0x5000000001017925 in string` 这样的一个错误，可以使用 Monkey 修改过的 class-dump

### Reveal

1. Mac 端下载 Reveal 打开，在 help 里找到 *Show Reveal Library in Finder -> iOS Library* ，打开要嵌入 iOS 应用的 framework
2. 打开该目录后，有一个 *RevealServer.framework*，将其中的 mach-o 文件 **RevealServer** 重命名为 `libReveal.dylib`。
3. 新建一个 `libReveal.plist` 文件，用 vim 写入要注入的 App 的 BundleId，如下所示：

```json
{
    Filter = {
    	Bundles = (
    		"tv.douyu.live"
    	);
	};
}

```

4. 将两个文件复制到手机的 `/Library/MobileSubstrate/DynamicLibraries` 目录下。
5. 重启应用，就会通过 Cydia 的 *MobileLoader*  加载该库了。

### Cycript

##### 如何用 Cycript 调试一个进程

1. `$ps -A`  获取所有运行的进程（一般在 `/var/mobile/Containers` 文件夹下，所以可以使用 `$ps -A | grep /var/mobile/Containers` 筛选更准确）
2. `cycript -p 进程名` 进入调试
3. **cmd+R** 清屏，**ctrl+D** 退出

#### Cycript 语法

```swift
// 获取 UIApplication
cy# UIApp    // 等价于 [UIApplication sharedApplication]

// 定义一个变量
cy# var rootViewController = UIApp.keyWindow.rootViewController
cy# var myView = [[UIView alloc] init]

// 找到内存中的该类型的对象
cy# choose(UIViewController)

// 通过 #+内存地址 获取对象
cy# #0x1234567.rootViewController

// 通过 *+对象 查看所有成员变量
cy# *UIApp
```
Cycript 主要用于查看控制器的层级、对象的成员变量以及动态调试界面

#### 编写 Cycript 库

编写 Cycript 库文件 `test.cy`:
```js
(function(exports) {
	// 递归打印 VC 层级
	ChildVcs = function(vc) {
		if (![vc isKindOfClass:[UIViewController class]]) throw new Error(invalidParamStr);
		return [vc _printHierarchy].toString();
	};

	// 递归打印view的层级结构
	Subviews = function(view) { 
		if (![view isKindOfClass:[UIView class]]) throw new Error(invalidParamStr);
		return view.recursiveDescription().toString(); 
	};
})(exports);
```
然后将文件放入 `usr/lib/cycript0.9` 文件夹下。
```js
cy# @import test
cy# ChildVcs(#0x12345678)
```