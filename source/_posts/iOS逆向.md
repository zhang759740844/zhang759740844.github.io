

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

Cydia Substrate  是一个框架，允许第三方开发者在越狱系统上打一些运行时的补丁和拓展一些方法，是**开发越狱插件的基石**。Cydia 自动安装了 Cydia Substrate，包含三个模块：

1. **MobileHooker：用于替换系统和应用的方法**。提供 `MSHookMessageEx` 和 `MSHookFunction` hook OC 和 C 函数，
2. **MobileLoader：用于将第三方动态库加载到运行的目标应用里**（注入 Reaveal 就是通过它）。首先通过环境变量 `DYLD_INSERT_LIBRARIES` 把自己加载到目标应用里，然后查找 `/Library/Mobile Substrate/DynamicLibraries/` 目录下所有的 plist 文件，如果 plist 文件的配置信息符合当前的应用，则通过 `dlopen` 函数打开对应的 dylib 文件
3. **Safe mode：如果插件导致 SpringBoard 崩溃，将会让设备进入安全模式，禁用所有的三方插件**

> 调试界面的 Reveal 就是通过 MobileLoader 动态加载的

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
6. 可以选择使用命令 `$otool -l {引用名}.decrypted | grep crypt` 查看加密标识，如果有输出 `cryptid 0` 标识该架构已经被解密了

> otool 可以用来查看 mach-o 的段信息，一样作用的还有 MachOView，是一个图形化的界面。

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

#### class-dump 原理

class-dump 主要是分析 mach-o 文件，进行**加载符号**，**解析协议列表**，**解析类列表**，**解析分类列表**

现在我们创建一个空的工程，build 之后通过 MachOView 对其进行分析。

##### 解析协议

首先通过 MachOView 找到 data 段中的 protolist。**除了各种 name 和代码之外，其他信息都是在 data 段中**：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho3.png?raw=true)

可以看到 `_objc_protolist` 段中包含两个协议信息。字段 offset 表示可执行文件加载到内存中后的相对偏移，**这个偏移是相对于 `__Text` 段的。**

再看字段 Data 的内容，它表示加载到内存中时候的虚拟地址。先简单罗列一下虚拟地址，实际地址，offset 这三者的关系：

> 虚拟地址 + ALSR = 实际地址
>
> `__Text` 的虚拟地址 + ALSR = `__Text` 的实际地址
>
> `__Text` 的虚拟地址 + 对象的 offset = 对象的虚拟地址
>
> `__Text` 的实际地址 + 对象的 offset = 对象的实际地址

因此，如果我们知道 `__Text` 的虚拟地址，就能知道 data 所表示的位置的 offset 了。`__Text` 段的虚拟地址的起始位置在 `__Text` 段中可以找到：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho4.png?raw=true)

> **因为在 `__Text` 段之前还有一个 `_PAGEZERO` 段，它的大小固定为 0x100000000**

因此可以计算出 data 的相对偏移为 8D30。我们来到了 __data 段：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho5.png?raw=true)

响应位置展示了一个协议完整的信息。8D30 保存的是 ISA 的信息，偏移8个字节的 8D38 保存的是协议名的信息，它指向的位置是 76B4:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho6.png?raw=true)

如此就可以到代码段中找到协议名的信息。

> 在 __data 段中查找的时候可以看到 MachOView 已经分析出了该协议的名词。整个从段信息到名词的过程就是上述的分析过程。

##### 类解析

类解析就不一步步分析了，直接看截图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho7.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho8.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho9.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho10.png?raw=true)

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

#### 如何用 Cycript 调试一个进程

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

> TODO： 增加获取属性和方法的相关库方法

#### Apple Configurator 2

#### 获取 ipa 包

我们可以通过 Apple Configurator 2 获取应用的 ipa 包。

手机连接上 Mac 之后，Apple Configurator 2 中会显示出手机的信息。这个时候点击添加应用，在 app store 中搜索你想要获取资源的引用：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/assets_1.png?raw=true)

搜索到之后点击下载。如果手机中没有目标应用，第一次就直接安装了，安装成功后再次搜索该应用下载。这个时候就会提示已经有该应用了，是否要替换：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/assets_2.png?raw=true)

这个时候不要做任何的点击。在终端中进入到 Apple Configurator 2 的缓存目录下：`~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps/` 你会发现目标应用的 ipa 包。

原因在于，Apple Configurator 2 下载的文件会临时保存在该临时目录下。如果安装完毕，就会立刻删除该 ipa 包。由于之前已经下载了该应用，Apple Configurator 2 弹窗提示是否需要替换，此时的 ipa 包还没来得及删除，可以复制出来。

#### 获取 Assets.car 中的资源文件

把 ipa 包复制出来后，使用解压工具对其进行解压。在 `payload` 文件夹下有资源包。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/assets_3.png?raw=true)

这个包里已经可以看到很多资源文件了，但是我们需要找到的是 `Assets.car` 这个压缩文件，这里面才是真正的我们需要的图片资源。这个`Assets.car`文件的解压需要用到[https://github.com/pcjbird/AssetsExtractor](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Fpcjbird%2FAssetsExtractor)使用起来非常方便。把 `Assets.car` 拖进该工具，指定输出目录就可以得到最终的全部资源文件

## 分析与调试

### 静态分析

#### hopper 使用

Mach-o 可以反编译为汇编代码，但是汇编代码无法完全反编译为 oc 代码。因为汇编操作的是寄存器，不同的类型以及不同的变量名的 oc 代码可能得到相同的汇编代码。

Hopper 和可以将 Mach-o 代码反编译为汇编代码、OC或者Swift伪代码。下载好之后，直接把 Mach-o 文件拖入 Hopper，即可开始分析。

### 动态调试

#### LLDB 调试

LLDB 通过 debugserver 和 app 通信。当 Xcode 调试手机时，Xcode 会将 debugserver 文件复制到手机中，以便在手机上启动一个服务，等待 Xcode 进行远程调试。只有设备连接到计算机真机调试时，debugserver 文件才会安装到设备的 `/Developer/user/bin` 目录下。 

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_1.png?raw=true)

但是默认情况下，*/Developer/usr/bin/debugserver* 缺少权限，只能使用 xcode 调试。因此，我们需要对 debugserver 重签名，获得两个权限： `get-task-allow` 和 `task_for_pid-allow`

如何给 debugserver 添加权限:

1. 将 debugserver 从目录拷贝到 mac。目录就是上述的 */Developer/usr/bin/debugserver* 
2. 使用 ldid 导出 debugserver 的权限到 debugserver.entilements 文件

ldid 是帮助修改 iPhone 上二进制授权文件的工具。通过 homebrew 安装：

```bash
brew install ldid
```

安装完后，导出 debugserver 的权限：

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
6. 手机端开启 server。开启调试后，进程会进入断电，被暂停：
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

##### debug 获取进程加载基地址

由于 ALSR 的原因，进程的虚拟地址不是从 0x0 开始的，会有一个随机偏移量。因此，当我们要读取运行的程序的内存的时候就需要知道这个 ALSR 的偏移量。

可以在 lldb 中通过 `image list -o -f` 拿到正在 debug 的进程的所有动态库的信息。其中第一项的第一列一般就是当前进程的基地址（不是ALSR地址）。

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
  - `watchpoint list`：列出所有内存断点
  - `watchpoint enable/disable/delete {断点编号}`：内存断点操作

  ```shell
  watch set variable self->age
  ```

  

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
3. 执行 `source ~/.zshrc` 重置 zshrc 配置。
4. 递归下载 theos `git clone --recursive https://github.com/theos/theos.git $THEOS`

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

相当于给原来的应用注入了一个动态库。可以在 `/Library/MobileSubstrate/DynamicLibraries` 中查看新安装的 `dylib` 和 `plist` 文件。

##### tweak 实现注意点

1. hook 的方法中的参数以及 self 一般都是 id 类型，所以不能用点语法，而要使用 get 方法：
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
6. 有时候调用原有方法或者 `%new` 的方法，有时会报 **instance method not found** 的错误，这需要我们再在实现的 xm 文件顶部声明一下该方法。类名任意，只要表示该方法声明过即可：
```objc
@interface {任意类名}
-(id){你的方法名}；
@end
```
7. 有时候使用某个类的时候还会报类不存在的错误。如果是使用自己创建的类直接 `#import "{类名}"` 即可。如果是被 hook 文件已经 import 的类，需要使用 `@class` 提前声明一下。
8. 可以通过关联对象的方式给实例添加属性
9. 如果要分多个文件编写，需要在 `makefile` 中配置相应文件，以空格分隔。针对文件量过多的情况，可以使用通配符表示一个文件夹内的文件。使用的时候直接直接 import，但是要把路径写完整，路径以 `Tweak.xm`  为基准。

```makefile
TWEAK_NAME = ${your tweak name}
${your project name}_FILES = ${以当前文件夹为基准的文件路径+文件名} ${以当前文件夹为基准的文件路径+文件名} ${以当前文件夹为基准的文件路径+文件名}
```

![makefile](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/tweak_1.png?raw=true)

 ![import](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/tweak_2.png?raw=true)

![通配符](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/tweak_3.png?raw=true)

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
$logify.pl xxx.h > {你想取的任意名字}.xm
```

通过修改 makefile 中的配置，将生成的 .xm 文件加入到编译文件中。直接添加到原来的 `Tweak.xm` 之后即可：

```makefile
{your project name}_FILES = Tweak.xm {你取的名字}.xm
```



但是经常会编译不过，需要手动修改：

1. 未定义头文件：报错 `unknown type name ‘XXX’`，在头部声明 `@class XXX;`,或者将 class 类型改为 id
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

#### MonkeyDev

MonkeyDev 的具体使用可以到 [MonkeyDev](https://github.com/AloneMonkey/MonkeyDev/wiki) Wiki 中查看

## 逆向进阶

### ASLR

Address space layout randomization 地址空间布局随机化

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_3.png?raw=true)

可以通过 lldb 的 `image list -o -f` 获得这个偏移地址:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_4.png?raw=true)

> image list -o -f 拿到的是减去 `__Text` 段基地址的偏移，即 ASLR
>
> image list 拿到的则是 `__Text` 的实际地址偏移，即基地址 + ASLR 地址

获得了偏移地址后，在通过 hopper 获得未使用 ALSR 的方法的地址，两者相加，就是该方法实际在内存中的地址了，可以为其设置断点：

```shell
breakpoint set -o {函数地址}+{偏移地址}
```

### 通用二进制文件

包含了多种架构的二进制文件叫做通用二进制文件，又叫 fat binary 胖二进制文件。
```shell
// 查看信息：
$file {文件名}
$lipo -info {文件名}

// 瘦身
$lipo {文件名} -thin armv7 -o {输出文件名}

// 合并
$lipo -create {文件名1} {文件名2} -o {输出文件名}
```

Xcode 中生成架构配置如图：
- Architecture： Xcode 支持的架构，不同 Xcode 版本不同
- Valid Architecture： 想要生成的架构

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_6.png?raw=true)

最终生成的架构就是支持的和想要的的交集。

### 程序加载

在编写一个程序时，看到的入口函数都是 main.m。实际上在 main 函数执行前已经执行了 `+load` 和 `constructor` 构造函数。现在要探讨在 main 函数执行前做了什么

#### dyld 简介

**系统内核在做好启动程序的准备工作之后就会从内核态切换到用户态，将工作交给 dyld。**

系统动态库会通过动态库加载器 dyld 被加载到内存中。为了优化程序启动速度，iOS 采用了共享缓存技术。在系统启动后被加载到内存中。当有新的程序加载时会先到共享缓存里寻找。找到就直接将共享缓存中的地址映射到目标进程的内存空间。 

#### dyld 加载流程

dyld 的时间线：

```
Load dylibs -> Rebase -> Bind -> ObjCruntime -> Initializers
```

1. dyld 从主执行文件的 header 获取到需要加载的所依赖动态库列表，并递归的将他们加载，为每一个动态库生成一个 ImageLoader 对象

   - 检查共享缓存是否映射到了共享区域

   - 加载所有通过 `DYLD_INSERT_LIBRARIES` 插入的库

2. 在加载所有的动态链接库之后，它们只是处在相互独立的状态，需要将它们绑定起来，这就是 Fix-ups

   - Rebasing：在 imageLoader 内部调整指针的指向，即修改 ASLR 带来的偏差
   - Binding：dylib 通过指针绑定会使用的外部的实例方法等符号的地址

3. ObjC Runtime 需要维护一张映射类名与类的全局表。当加载一个 dylib 时，其定义的所有的类都需要被注册到这个全局表中。

4. 执行初始化方法，`+load` 和 `constructor`  就是在这里执行的

5. 通过 Load Command 找到 `LC_MAIN` 即 main 函数位置，执行

### Mach-O 文件

#### MachO 文件基本构成

Mach-O 是苹果的可执行文件，结构由三部分组成：
- Header：文件类型，目标架构类型
- Load Commands：描述载入内存的有哪些段，段有多大，从哪里开始
- Data：段的数据

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho0.png?raw=true)

使用 MachOView 查看，我们可以看到有各种各样的段。class-dump 就是通过这种方式获取到头文件信息的。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho1.png?raw=true)

具体看一下 Macho 的 Load Command，它包含了 Macho 中各个段的基本信息：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho_10.png?raw=true)

一般可执行文件会分为许多个 section，section 又会根据权限的不同整合为多个 fragment

，一般分为四个 fragment：

- `__PAGEZERO` 空指针陷阱段，映射到虚拟内存空间的第一页，用于捕捉对 NULL 指针的引用
- `__TEXT` 代码段，只读，包括函数，和只读的字符串(如  `__TEXT,__text` 保存所有代码，又如 `__TEXT.__objc_classname` 保存 Objective-C 类名称)__
- `__DATA` 数据段，读写，包括可读写的全局变量等(如 `__DATA,__data` 保存初始化过的可变数据，又如 `__DATA.__objc_classlist` 保存所有类实体的指针，指向 __data 中保存的 objc_class 实例）
- `__LINKEDIT`动态链接器需要使用的信息，包括重定位信息，绑定信息，懒加载信息等。

#### 代码段和数据段的具体组成

__TEXT 包含以下 section：

- __text：程序可执行代码区域
- __stubs：简介符号存根，跳转到懒加载指针表
- __stub_helper：帮助解决懒加载符号加载的辅助函数
- __objc_methname：方法名
- __objc_classname：类名
- __objc_methtype：方法签名
- __cstring：c风格字符串

__DATA 包含以下 section：

- __nl_symbol_ptr：非懒加载指针表，在 dyld 加载时会立即绑定
- __la_symbol_ptr：懒加载指针表，第一次调用才绑定值
- __got：非懒加载全局指针表
- __mod_init_func：constructor 函数
- __mod_term_func： destructor 函数
- __cfstring：OC 字符串
- __objc_classlist：程序中类的列表
- __objc_nlclslist：程序中实现了 +load 方法的类
- __objc_protolist：协议的列表
- __objc_classrefs：被应用的类列表



#### 懒加载和非懒加载

在之前说的dyld执行流程中有一环叫做 binding，即绑定符号的地址。iOS 系统为了加快系统启动速度，将符号分成了懒加载符号和非懒加载符号。

非懒加载符号在 dyld 加载时就会绑定真实的值。懒加载符号不会，只有第一次去调用它时才会绑定真实的地址，第二次调用直接使用真实地址。

如系统方法 print 的绑定过程如下图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/bind_1.jpeg?raw=true)

### ARM 汇编

#### ARM寄存器

X0-X30 是 31 个通用寄存器，低 32 位用 W0-W30 表示。X30 是一个特殊寄存器，用于保存函数调用完成时的返回地址。

SP 为堆栈指针寄存器，通过 X31 寄存器访问

PC 为保存当前指令地址的程序计数器

LR 指向返回地址，对应于 X30

FP 指向栈帧的底部，对应于 X29（即 Base Pointer）



#### 常见指令

`ldr Xn, addr`: 从 addr 读取内容存到 Xn 中

`str Xn, addr`: 将 Xn 写入 addr 指向的内存

`cbz Xn, label` : 如果 Xn 为 0，则跳转到 label

`cbnz Xn, label`: 如果 Xn 不为 0，跳转到 label

`bl label`: 无条件跳转，会将下一条指令地址写到 X30 处

#### 堆栈调用

堆栈调用过程如下：

1. 函数调用前：
   1. 开辟堆栈控件
   2. 保存 FP 和 LR 寄存器，以便找到上一个栈帧和返回地址
   3. 设置新的 FP 寄存器
   4. 保存子函数会用到的寄存器
   5. 保存局部变量或参数
2. 函数调用结束：
   1. 还原 FP 和 LR 寄存器
   2. 释放栈帧空间
   3. 跳到 LR 继续执行

参数一般通过 X0-X7传递，当参数小于 8 个时候，默认是直接存到寄存器中的。但是如果超过 8 个，那么会在方法调用前，将多出的那个存入堆栈中，子方法需要到堆栈中取值。

返回结果一般通过 X0 传递

一个方法的汇编的例子：

```c
void fooFp() {
    int a = 4;
    int b = 5;
    fooFp2();
}

void fooFp2() {
    int a = 2;
    int b = 3;
}
```

```assembly
ArmAssembly`fooFp:
 0x100dbe76c <+0>:  sub    sp, sp, #0x20             ; 申请栈空间
 0x100dbe770 <+4>:  stp    x29, x30, [sp, #0x10]     ; 保护寄存器的值
 0x100dbe774 <+8>:  add    x29, sp, #0x10            ; 改变fp寄存器的值，用于执行栈底
 
 0x100dbe778 <+12>: mov    w8, #0x5
 0x100dbe77c <+16>: orr    w9, wzr, #0x4
 0x100dbe780 <+20>: stur   w9, [x29, #-0x4]
 0x100dbe784 <+24>: str    w8, [sp, #0x8]
 0x100dbe788 <+28>: bl     0x100dbe798               ; fooFp2 at ViewController.m:154
 
 0x100dbe78c <+32>: ldp    x29, x30, [sp, #0x10]     ; 恢复之前保存的fp和lr的值
 0x100dbe790 <+36>: add    sp, sp, #0x20             ; 恢复sp指针
 0x100dbe794 <+40>: ret 
 
```



### hook

#### fishhook

苹果为了能在 Mach-O 文件中访问外部函数，采用了一个技术，叫做PIC（位置代码独立）技术。当你的应用程序想要调用 Mach-O 文件外部的函数的时候，或者说如果 Mach-O 内部需要调用系统的库函数时，Mach-O 文件会：

1. 先在 Mach-O 文件的 _DATA 段中建立一个指针ptr（8字节的数据，放的全是0），这个指针变量指向外部函数。
2. DYLD 会动态的进行绑定！将 Mach-O 中的 _DATA 段中的指针，指向外部函数。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/macho2.png?raw=true)

上图中可以看到，其他部分都是实实在在的代码信息，没有好的修改办法。只有红色框框中的 `nl_symbol_ptr` 和 `la_symbol_ptr`是指针段，由于指针都是长度固定的，所以方便修改指针地址，进而达到 hook 的目的。

所以说，C的底层也有动态的表现。C在内部函数的时候是静态的，在编译后，函数的内存地址就确定了。但是，外部的函数是不能确定的，也就是说C的底层也有动态的。fishhook 之所以能 hook C函数，是利用了 Mach-O 文件的 PIC 技术特点。也就造就了静态语言C也有动态的部分，通过 DYLD 进行动态绑定的时候做了手脚。

**我们经常说符号，其实 _DATA 段中建立的指针就是符号。fishhook的原理其实就是，将指向系统方法（外部函数）的符号重新进行绑定指向内部的函数。这样就把系统方法与自己定义的方法进行了交换。这也就是为什么C的内部函数修改不了，自定义的函数修改不了，只能修改 Mach-O 外部的函数。**

### iOS 签名

签名过程如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/逆向_5.png?raw=true)

当我们修改了别人的 APP 之后，我们就需要将修改过的 ipa 重签名，才能安装到手机上。重签名了的应用可以安装到未越狱手机。

### IPAPatch 免越狱调试 APP

IPAPatch 就是通过从签名达到免越狱注入代码的目的。使用起来非常简单。

1. 使用 PP 助手下载一个已越狱的应用的 ipa
2. 下载 IPAPatch 的工程
3. 使用已越狱的 ipa 替换工程中 Assets 文件夹下的 **app.ipa** 文件。注意，名字要改为 app.ipa
4. 将 RevealServer.framework 放置在 Assets/Frameworks/RevealServer.framework
5. 修改 bundleId
6. Run Xcode 安装到手机

## 安全保护

### 静态混淆

#### 宏定义

使用宏将方法属性名修改为其他无意义的字符串

### 动态保护

#### 反调试

**ptrace**

UNIX 早期版本提供的一种对运行中的进程进行跟踪和控制的手段，就是系统调用 ptrace。通过 ptrace 实现对另一个进程的调试跟踪

可以通过参数 `PT_DENY_ATTACH` 禁用调试。这个参数告诉系统阻止调试器依附。所以，最常用的反调试方法就是通过调用 ptrace 来实现反调试。

**sysctl**

当一个进程被调试时，该进程中的一个标志位用来标记正在被调试。可以定时通过 sysctl 查看这个标志位

#### 反反调试

hook函数 -> 判断参数 -> 返回结果

越狱设备直接可以hook以上说到的反调试函数。非越狱设备可以通过 fishhook，hook 反调试函数。

#### 反注入

可以定期调用 `_dyld_get_image_name()` 方法，获取正在加载的动态库名，比较是否是白名单内的动态库名来实现注入检测。

#### hook 检测

hook 包括 Method Swizzle，符号表替换，inline hook 等。不同的 hook 方式，需要制定不同的检测方案

**Method Swizzle**

Method Swizzle 的原理是替换 imp，通过 dladdr 得到 imp 所在的模块，判断模块是不是主二进制模块，如果不是就是被 hook 了。

**符号表替换**

fishhook 是基于懒加载符号表和非懒加载符号表进行替换的，所以遍历符号表中的指针就能判断程序是否被恶意 hook 了。

非懒加载的指针指向真实地址，懒加载的指针在没有解析到真正的地址钱指向 `__stub_helper`，所以遍历符号表，判断是否指向了系统模块或者 `__stub_helper` 即可。

#### 完整性校验

逆向过程设计到对文件 load command 的修改，对文件进行重签名，修改 BundleId。可以从上述几个方面校验

**load command**

直接读取 Mach-O load command 中的 `LC_LOAD_DYLIB` ，判断是否有非白名单动态库

**代码校验**

获取内存中代码的 MD5 值，如果代码修改了，就会不一样

**重签名校验** 

判断 bundle ID 是否被修改

## 8086简介

8086 是 x86系列处理器的开端，所以后面用 x86 代替 32位处理器。

### CPU 的组成

CPU 的三大组成：

- 运算单元
- 数据单元
- 控制单元

运算单元做加法或者位移的操作。

数据单元包含CPU内部的缓存和寄存器组。

控制单元可以获得下一条指令，然后执行。这个指令会指导运算单元取出数据单元中的某几个数据，计算出结果，然后放到数据单元的某个地方。

### 8086 的寻址方式

8086 的总线是有16根，但是可以寻址的范围为 2^20 byte。

20位的物理地址 = 16位的段地址 * 16 + 4位的偏移地址

### CPU 中的数据单元

8086 CPU 中的数据单元如下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/8086_1.png?raw=true)

其中：

- AX，BX，CX，DX 为数据寄存器，存放操作的数据
- CS 代表代码段的起始地址。IP 表示偏移地址。每读取一条指令，**IP=IP + 所读取指令的长度**。
- SS 表示栈的起始位置。SP 表示栈的偏移地址。BP 是入参和临时变量的分界，通过 BP 及偏移量拿到入参和临时变量。
