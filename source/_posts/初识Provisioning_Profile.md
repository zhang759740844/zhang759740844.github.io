title: 初识Provisioning Profile
date: 2016/8/26 14:07:12  
categories: IOS
tags: [Xcode]

---

真机调试的时候遇到了*provisioning profile即将过期*的警告，于是搜索并对provisioning profile作了一定了解。结合[关于Certificate、Provisioning Profile、App ID的介绍及其之间的关系](http://www.cnblogs.com/cywin888/p/3263027.html)进行梳理。

<!--more-->

## 基本概念
### Certificate
证书是用来给应用程序签名的，只有经过签名的应用程序才能保证他的来源是可信任的，并且代码是完整的， 未经修改的。在Xcode Build Setting(点击工程文件)的Code Signing Identity中，你可以设置用于为代码签名的证书。

众所周知，我们申请一个Certificate之前，需要先申请一个Certificate Signing Request (CSR) 文件，而这个过程中实际上是生成了一对公钥和私钥，保存在你Mac的Keychain中。代码签名正是使用这种基于非对称秘钥的加密方式，用私钥进行签名，用公钥进行验证。如下图所示，在你Mac的keychain的login中存储着相关的公钥和私钥，而证书中包含了公钥。你只能用私钥来进行签名，所以如果没有了私钥，就意味着你不能进行签名了，所以就无法使用这个证书了，此时你只能revoke之前的证书再申请一个。因此在申请完证书时，最好导出并保存好你的私钥。当你想与其他人或其他设备共享证书时，把私钥传给它就可以了。私钥保存在你的Mac中，而苹果生成的Certificate中包含了公钥。当你用自己的私钥对代码签名后，苹果就可以用证书中的公钥来进行验证，确保是你对代码进行了签名，而不是别人冒充你，同时也确保代码的完整性等。 

![证书的私钥和公钥](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/profile_certificate.png?raw=true)

证书主要分为两类：Development和Production，Development证书用来开发和调试应用程序，Production主要用来分发（生产，上架Appstore）应用程序（根据证书种类有不同作用）

### APP ID
App ID用于标识一个或者一组App，App ID应该是和Xcode中的Bundle ID是一致的或者匹配的。App ID主要有以下两种：
- Explicit App ID：唯一的App ID，这种App ID用于唯一标识一个应用程序，例如com.ABC.demo1，标识Bundle ID为com.ABC.demo1的程序。
- Wildcard App ID：通配符App ID，用于标识一组应用程序。例如\*可以表示所有应用程序，而com.ABC.*可以表示以com.ABC开头的所有应用程序。

每创建一个App ID，我们都可以设置该App ID所使用的APP Services，也就是其所使用的额外服务。每种额外服务都有着不同的要求，例如，如果要使用Apple Push Notification Services，则必须是一个explicit App ID，以便能唯一标识一个应用程序。

### Device
Device最简单了，就是iOS设备。Devices中包含了该账户中所有可用于开发和测试的设备。 每台设备使用UDID来唯一标识。

每个账户中的设备数量限制是100个。Disable 一台设备也不会增加名额，只能在membership year 开始的时候才能通过删除设备来增加名额。

### Provisioning Profile
一个Provisioning Profile文件包含了上述的所有内容：证书、App ID、设备。

试想一下，如果我们要打包或者在真机上运行一个应用程序，我们首先需要证书来进行签名，用来标识这个应用程序是合法的、安全的、完整的等等；然后需要指明它的App ID，并且验证Bundle ID是否与其一致；再次，如果是真机调试，需要确认这台设备能否用来运行程序。而Provisioning Profile就把这些信息全部打包在一起，方便我们在调试和发布程序打包时使用，这样我们只要在不同的情况下选择不同的profile文件就可以了。而且这个Provisioning Profile文件会在打包时嵌入.ipa的包里。

例如，如下图所示，一个用于Development的Provisioning Profile中包含了该Provisioning Profile对应的App ID，可使用的证书和设备。这意味着使用这个Provisioning Profile打包程序必须拥有相应的证书，并且是将App ID对应的程序运行到Devices中包含的设备上去。

![Provisioning Profile组成](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/Provisioning_Profile_consist.png?raw=true)

如上所述，在一台设备上运行应用程序的过程如下：

![Provisioning Profile验证](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/Provisioning_Profile_Confirm.png?raw=true)

与证书一样，Provisioning Profile也分为Development和Distribution两种

## Xcode 7+ 免证书真机调试
在Xcode 7+ 中，苹果改变了自己在许可权限上的策略，此前Xcode只开放给注册开发者下载，但Xcode 7改变了这种惯有的做法，无需注册开发者账号，仅使用普通的Apple ID就能下载和上手体验。此前开发者需每年支付99美元的费用成为注册开发者才能在iPhone、iPad、Apple Watch等真机上运行代码，苹果新的开发者计划则放宽要求，无需购买，只要你感兴趣同样可以在真机设备上测试app

### 使用方法
1. 打开自己已有的工程或者打开Xcode新建一个简单工程
2. 菜单栏选择Xcode,下拉菜单中选择Preferences...(快捷键 command + , )
	![1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NoCertificateDebug.png?raw=true)
3. 在Accounts选项卡添加自己的Apple ID。添加完成后你会看到这样的信息。可以看到下面显示了iOS和Mac的Free标记了，在这之前是没有这些标记的。
	![2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NoCertificateDebg2.png?raw=true)
4. 将测试的iPhone或者其他真机测试设备连接Mac，Xcode选中真机测试设备，直接点击运行。
	![3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NoCertificateDebg3.png?raw=true)
5. Xcode报错如下图所示，直接点击Fix Issue。
	![4](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NoCertificateDebg4.png?raw=true)
6. 警告也没了，证书也生成了，重新点击run按钮，可以开始愉快的玩耍了😁。
	![5](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NoCertificateDebg5.png?raw=true)



