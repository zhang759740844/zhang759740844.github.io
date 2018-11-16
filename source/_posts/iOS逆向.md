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
```shell
cy# @import test
cy# ChildVcs(#0x12345678)
```

## 分析与调试

### 静态分析

#### hopper 使用

Mach-o 可以反编译为汇编代码，但是汇编代码无法完全反编译为 oc 代码。因为汇编操作的是寄存器，不同的类型以及不同的变量名的 oc 代码可能得到相同的汇编代码。

Hopper 和可以将 Mach-o 代码反编译为汇编代码、OC或者Swift伪代码。下载好之后，直接把 Mach-o 文件拖入 Hopper，即可开始分析。

### 动态调试

#### LLDB 调试

LLDB 通过 debugserver 和 app 通信

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_1.png?raw=true)

默认情况下，*/Devloper/usr/bin/debugserver* 缺少权限，只能使用 xcode 调试。因此，我们需要对 debugserver 重签名，获得两个权限： `get-task-allow` 和 `task_for_pid-allow`

如何给 debugserver 添加权限:

1. 将 debugserver 从目录拷贝到 mac
2. 使用 ldid 导出 debugserver 的权限到 debugserver.entilements 文件

```shell
$ldid -e debugserver > debugserver.entilements
```

3. debugserver.entilements 文件是个配置文件，打开并添加这两个权限：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_2.png?raw=true)

4. 使用 ldid 重签名：
```shell
$ldid -S debugserver.entilements debugserver
```
（上面的 ldid 的重签名也可以使用 codesign 代替）
```shell
# 查看 （在终端中显示，可以复制保存为 debugserver.entitlements）
$codesign -d --entitlements -debugserver
# 签名
$codesign -f -s - --entitlements debugserver.entitlements debugserver
```
5. 将重签名后的 debugserver 拖到手机 *device/usr/bin* 目录下，这样可以直接使用该命令
6. 手机端开启 server：
```shell
$debugserver *:{任意端口，如10010} -a {进程名}
```
7. Mac 端启动 lldb
```shell
# 进入lldb 模式
$lldb
# 连接，连接需要一段时间
(lldb) process connect connect://{手机IP}:{前面启动时的端口号}
# 连接成功自动进入断点，需要继续运行app
(lldb) c
```

#### lldb 使用

- 打印相关：
  - `p {指令}`：执行指令
  - `bt`：打印调用堆栈 backtrace
  - `frame variable`：打印当前栈帧的变量
- 调试相关：
  - `c`：继续执行
  - `n`：单步运行
  - `step`：step-in
  - `finish`：step-over
- 代码断点相关
  - `breakpoint set -n {方法名}`：为某个方法设置断点。
```shell
# 更高级的，设置某一个类的某一个方法的断点，需要使用引号包裹
breakpoint set -n "-[ViewController touchesBegan:withEvent:]"
```
	- `breakpoint list`：列出所有断点，包含编号
	- `breakpoint enable/disable/delete {断点编号}`：断点操作
- 内存断点相关
  - `watchpoint set variable {变量名}`：变量值变化的时候触发
```shell
watch set variable self->age
```
	- `watchpoint list`：列出所有内存断点
	- `watchpoint enable/disable/delete {断点编号}`：内存断点操作
#### theos

##### 下载 theos

1. 安装签名工具 `brew install ldid`
2. 修改 *.zshrc*
```bash
# 这样可以用 $THEOS 代替 ~/theos
export THEOS=~/theos
# 修改环境变量,这样就可以在任意位置执行 `~/theos/bin` 路径下命令了
export PATH=~/theos/bin:$PATH
```
3. 递归下载 theos `git clone --recursive https://github.com/theos/theos.git $THEOS`

##### 使用 theos 创建 tweak

1. 执行下载下来的命令 `$nic.pl`
2. 根据提示选择要创建 *iphone/tweak*，然后设置自己的项目名，bundleID，还有目标应用的 bundle identifier（用 cycript 获取，`[NSBundle mainBundle].bundleIdentifier`)，生成一系列文件。
3. 编辑生成的 *Makefile* 文件，在文件最上方添加配置：
```makefile
# 设置IP和端口号，表示要通过 SSH 的方式安装这个 theos
# 切记一定要放在最上面
export THEOS_DEVICE_IP={你的手机的IP}
export THEOS_DEVICE_PORT=22

# ... 省略了自动生成的配置部分
```
4. 编写代码，编辑 *Tewak.xm* 文件：
```objc
%hook {要hook的类名}
{根据 class-dump 找到感兴趣的要hook的方法}
%end
```
5. 命令行执行：
```shell
# 编译
$make
# 打包
$make package
# 安装 会重启 SpringBoard
$make install
```

##### tweak 实现注意点

1. hook 的方法中的参数以及 self 一般都是 id 类型，所以不能用ongoing点语法，而要使用 get 方法：
```objc
- (id)tableView:(id)tableView cellForRowAtIndexPath:(id)indexPath {
	int num = [indexPath section];
}
```
2. 可以使用宏，在文件顶部定义：
```objc
#define SomeMethod [NSUserDefaults standardUserDefaults]
```
3. hook 方法默认是 hook 已有方法，如果增加的方法是原来没有的需要加上 `%new`:
```objc
%new -(void)someFunc:(UIButton *)button {
	...
}
```
4. hook 方法要调用原来的实现方法，使用 `%orig` 替代：
```objc
- (long long)numberOfSectionsInTableView:(id)tableView {
	return %orig + 2
}
```
5. 资源文件放在新建的 tweak 生成的 *layout* 文件夹下，该目录对应的就是手机的根目录，可以自己在 layout 下创建文件夹层级，代码中引用：
```objc
UIImage *myImage = [UIImage imageWithCOntentOfFile:@"/{自己创建的文件夹层级}/{文件名}"]
```
6. 有时候调用 get 方法或者 `%new` 的方法，有时会报 **instance method not found** 的错误，这需要我们再在实现的 xm 文件顶部声明一下该类或者方法：
```objc
@interface {类名}
-(id){你的方法名}；
@end
```
7. 有时候使用某个类的时候还会报类不存在的错误。如果是使用自己创建的类直接 `#import "{类名}"` 即可。如果是被 hook 文件已经 import 的类，需要使用 `@class` 提前声明一下。

##### Tweak 实现原理

- 生成的插件会被安装在 */Library/MobileSubstrate/DynamicLibraries* 中。生成的文件包括 **.dylib** 包含编译后的 tweak 代码，和 **.plist** 存放着 hook 的目标 APPID
- 打开 App 时，Cydia Substrate 会去加载对应的 dylib，修改内存中的代码逻辑， 执行 dylib 中的函数
- tweak 不会修改可执行文件，仅仅只是修改了内存的逻辑。所以 tweak 可以对未砸壳 App 修改，但是必须要使用越狱手机。

##### Logos 语法

- `%hook` `%end` hook一个类的开始和结束
- `%log;` 打印方法调用详情，在 Xcode 的日志输出中查看
- `%c({类名})` 获取类对象，相当于 `[xxx class]`，直接调用可能有错
- `%orig` 函数原来的代码逻辑
- `%new` 添加一个新的方法

##### logify.pl

可以将头文件快速转换成已经包含打印信息的 xm 文件（自动添加 `%log` 语句）：
```shell
$logify.pl xxx.h > Tweak.xm
```

但是经常会编译不过，需要手动修改：
1. 未定义头文件：报错 `unknown type name ‘XXX’`，在头部声明 `@class XXX;`
2. 未声明协议：报错`no type or protocol named 'XXX'`，在头部声明 `@protocol XXX`
3. 不能存在 weak：报错 `cannot create __weak reference`，替换掉所有的 `__weak` 为空字符串
4. 不能存在非 oc 方法：报错 `expected selector for Objective-C method` ，删除以点开头的非 OC 的方法
5. 带协议的参数报错：报错 `cast from pointer to smaller type 'unsigned int' loses information`。如果有一个参数类型遵循某个协议，那么 `%log` 就无法通过，需要把协议删除。
6. `HBLogDebug` 类型错误：报错 `cast from pointer to smaller type 'unsigned int' loses information`，`HBLogDebug` 本身是用来打印方法返回值的，有一些地方返回值 id 类型，会被转化为 unsigned int 类型，因此报错。需要替换：
```objc
// 原始语句
HBLogDebug(@"=0x%x", (unsigned int)r);
// 批量替换为
HBLogDebug(@"=0x%@", r);
```



## 逆向进阶

### 程序加载

系统动态库会通过动态库加载器 dyld 被加载到内存中。为了优化程序启动速度，iOS 采用了共享缓存技术。在系统启动后被加载到内存中。当有新的程序加载时会先到共享缓存里寻找。找到就直接将共享缓存中的地址映射到目标进程的内存空间。