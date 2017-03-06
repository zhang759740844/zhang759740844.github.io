title: iOS 证书相关
date: 2017/3/4 16:07:12  
categories: 计算机
tags:

	- Xcode
---

关于苹果的证书问题搞得焦头烂额，代码没问题，跑不起来这是最气的。所以决定完整地了解一下。

<!--more-->

## 基础

### 密码基础

加密方法可以分为两大类。一类是**单钥加密**（private key cryptography），还有一类叫做**双钥加密**（public key cryptography）。前者的加密和解密过程都用同一套密码，后者的加密和解密过程用的是两套密码。

单钥加密情况下，密钥只有一把，加密解密都通过其完成。密钥的保存非常重要。

双钥加密时，密钥有两把，一把给别人使用的公钥，一把只有自己知道的私钥。公钥和私钥一一对应，公钥能解开私钥加密的信息，私钥也能解开公钥加密的信息。同时生成公钥和私钥比较容易，但是从公钥推算出私钥是几乎不可能的。

目前，通用的单钥加密算法为DES（Data Encryption Standard），通用的双钥加密算法为RSA（ Rivest-Shamir-Adleman）。

在双钥体系中，**公钥用来加密信息（因为只有私钥才能打开），私钥用来数字签名（保证数据完整性的，但它不保证数据加密，不保证数据传输途中无人嗅探窃听）**。

因为任何人都可以生成自己的（公钥，私钥）对，所以为了防止有人散布伪造的公钥骗取信任，就需要一个可靠的第三方机构来生成经过认证的（公钥，私钥）对。

### 双钥加密原理

关于双钥加密，在之前写过的 “http原理” 一文中也有大概的梳理。这边再仔细说下。

需要明确：公钥任何人都能有，私钥只能自己持有，如果别人拿到了你的私钥，那么别人也就成为了你。因此，用公钥加密，私钥解密的信息是绝密的；而用私钥加密，公钥解密的信息，并不能保证安全。所以用私钥加密的信息一般用来保证传输信息的完整性与未被修改。

整个加密解密的流程大概是这样的：

A 有公钥，B 有私钥。A 将用公钥加密的消息发送给 B，那么只有拥有私钥的 B 才能够解密，其他人无法解密出消息明文。

B 要发消息给 A。由于公钥是公开的，那么 B 用私钥加密的消息对所有人其实都是公开的，所以一般不用 B 的私钥加密消息。而只是拿 B 的私钥产生一个“数字签名”。数字签名就是 将 B 即将给 A 发送的消息的 hash 值（通过 MD5、SHA等算法）进行加密。B 发送消息的时候，将消息原文和数字签名一并发出。A 接收到两者后，对原文取 hash，与公钥解密的数字签名中获得的 hash 值比对，如果相同表示消息没有经过别人的修改。

上面保证了 A 与 B 能安全的互相通信。但是没有解决“如何才能使 B 给 A 发送的消息不被别人窥探？” 很容易想到，只要再来一套公钥私钥，即 B 同时也拥有 A 的公钥，发信息的时候用 A 的公钥加密即可，不过这样处理非常麻烦费时。https 是这样做的：客户端拥有服务端的公钥（即上面所说的 A，先不要管怎么拿到的服务端公钥的以及是如何确定就是服务端公钥而不是其他攻击者的公钥的这两点），服务端拥有其自己的私钥（即 B）。A 在本地生成一段随机字符，作为之后单钥加密的密钥，将这个密钥通过公钥加密发给 B，这样 A B 就共同拥有了其他人都不知道的共同密钥，两者之间的通信就安全了。

上面说到如何拿到服务端的公钥，在客户端发起请求后，服务端会返回一个证书，这个**证书中就包含着公钥**。

客户端如何相信这个证书就是想要请求的服务端的证书而不是攻击者的证书呢？这就需要一个权威的第三方的验证，即"证书中心"（certificate authority，简称CA），为公钥做认证。证书中心用自己的私钥为服务器的公钥加密，连同自身信息以及服务端的信息保存在新生成的"数字证书"中。客户端最终获取到证书就是经过证书中心加密过的证书。拿到证书后，客户端先到自身根证书列表查看该证书中心是否是本机信任的证书中心（自己计算机的物理安全是前提，如果别人已经可以直接控制你的计算机，修改根证书列表，那什么证书安全也救不了你。）。然后就用根证书里的公钥去解析该证书，获得服务端的公钥。

其实道理就是：你不是不相信发来的公钥的可靠性么。那么就你自己先保存一些信任的第三方的公钥，通过 CA 的公钥去解析获得服务端的公钥。攻击者是没有 CA 认证过 ，或者即使认证了，证书中的信息也是与真正要请求的服务端不同的。因此，这样就能保证其可信性了。

说完了 https，其实 SSH 也是类似的，服务端传输公钥给客户端给其加密使用，同样的也会遇到中间人攻击的问题。由于 SSH 没有 CA 认证，所以在第一次连接服务器时要慎重，会提示你是否接受这个远程主机的公钥：

```
Are you sure you want to continue connecting (yes/no)? yes
```

如果接受，那么该公钥就会保存在本机中。下次再连接这台主机，系统就会认出它的公钥已经保存在本地了。

https 和 SSH 都接受用户名密码登录，即客户端通过服务器下发的公钥加密后传输给远程主机。SSH 还接受公钥登录，就是本机生成一对公钥和私钥（保存在根目录的 .ssh 文件夹下），将公钥保存在服务器上，有点类似于上面说的 A 和 B 各自保存自己的公钥和对方的私钥。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

**在 Apple 开发网站上传包含公钥的 CSR 文件作为换取证书的凭证（Upload CSR file to generate your certificate），有点类似为github账号添加SSH公钥到服务器上进行授权。** 

## iOS 相关

### App IDs

app id 和 bundle identifier 是一致（Explicit）的或匹配（Wildcard）的。

在 name 中输入 app id 显示的名字：

![app id name](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_appid.png?raw=true)

可以选择 explicit app id 或者 wildcard app id。前者用于表示标识唯一的应用程序，后者为含有通配符的 app id，用于标识一组应用程序。一般推荐用前者。

![app id bundleid](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_bundleid.png?raw=true)

最后勾选需要用到的 app service，比如 push notification:

![app id app service](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_pushnotification.png?raw=true)

继续点击 continue->submit->done，完成 app id 的创建。可以点击刚申请的 id，对其进行编辑：

![app id app edit](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_edit.png?raw=true)

比如如果用到推送功能，就要为其添加推送证书：

![app id app edit](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_push.png?raw=true)

总结：app id 主要设置了 bundle id 的 app 特有标识，另外为该 app 设置了所需的服务。



### Certificates

iOS证书是用来证明iOS App内容（executable code）的合法性和完整性的数字证书。对于想安装到真机或发布到AppStore的应用程序（App），只有经过签名验证（Signature Validated）才能确保来源可信，并且保证App内容是完整、未经篡改的。上面讲了许多关于证书的原理，这里主要讲讲 iOS 中证书的配置。

iOS证书分两种：**开发证书（Development）** 和**生产证书（Production）**。

- 开发证书用于开发和调试应用程序，可用于联机调试。无论是 debug 还是 release 用的都是开发证书
- 生产证书用来发布应用程序。

#### 生成证书请求文件 （CSR）

CSR 用来生成公钥和私钥，是制作 iOS 证书必须的一样东西。通过如下方式申请：

![app id app csr](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_csr.png?raw=true)

选择 “从证书颁发机构请求证书” 后出现下面情况，填写开发账号右键和常用名称，CA 电子右键地址可不填，勾选 “存储到磁盘”，然后继续：

![app id app info](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_info.png?raw=true)

最后得到一个 csr 文件

![app id csr](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_csrcomplete.png?raw=true)

#### 申请证书

现在进入申请证书的流程：

![apply certificate](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_apply.png?raw=true)

由于 apple 规定一个账号只能申请3个开发证书，2个发布证书，所以现在图片上的部分不能被勾选。除了开发证书和发布证书，还可以勾选 `Apple Push Notification service SSL (Sandbox)` 和 `Apple Push Notification service SSL (Sandbox & Production)` 来制作开发和发布推送证书。

如果是制作推送证书，需要绑定 App id：

![notification_certificate](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/certificate_push.png?raw=true)

如果不是推送证书跳过上一步，直接往下走，上传 CSR 文件：

![csr_certificate](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/certificate_csr.png?raw=true)

点击生成，就完成了证书的创建。将生成的证书下载后双击导入钥匙串。在钥匙串的我的证书中就能看到相应证书，这个证书是包含公钥和私钥的。

#### 导出证书

对于团队合作，别人需要你创建的证书，你就要将证书（包括公钥和私钥）导出为 .p12 格式的文件，发给别人使用。注意，在苹果开发者网站上的证书只有公钥没有私钥。别人即使下载下来，没有私钥也是没用的。

### Devices

Device是指运行iOS系统用于开发调试App的设备（即苹果设备）。每台Apple设备使用UDID来唯一标识。

设备的UDID可通过iTunes->Summary或者Xcode->Window->Devices获取。

添加设备比较简单，就是输入设备的 UUID，并且给设备取一个别名即可。

### Provisioning Profiles

Provisioning Profile 包含了上述的所有内容：证书，app id，设备。用来将证书和 app id 绑定。

Provisioning Profile 也分为 Development 和 Distribution 两类，有效期同 Certificate 一样。**Development 版本的 ProvisioningProfile 用于开发调试，包括 debug 和 release 版本**，Distribution 版本的 ProvisioningProfile 只用于提交 App Store 审核，其不指定开发测试的 Devices。

在 XcodeTarget->Build Settings->Code Signing->Provisioning Profile 可选择“Automatic”，xcode 会根据该 Target 的“Bundle identifier”选择默认的配置文件及证书：

![provision profile](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_profile.png?raw=true)

自动签名的情况下，不需要自己设置 Provisioning Profiles。不过你也可以选择手动设置签名：

![provision profile](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_manual.png?raw=true)

debug 和 release 用的都是开发的配置文件。

### 常遇的一些问题

#### 证书过期

有时候编译通过，但是要装进测试机里时就会报如下的错误：

![证书过期](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_revoke.png?raw=true)

如果你确信不是你证书真的过期了的话，可以通过 Xcode->preferences->account->ViewDetails->reset 来重置你 Xcode 中的证书信息：

![证书过期](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_reset.png?raw=true)

重置好后一般就可以正常安装了。

在这个界面下，你还可以管理你的 Provisioning Profiles，可以手动下载所需文件，以及删除多余文件。不用从官网下载好后导入，更不用到文件夹里一个个比对删除了。



更多这方面的信息可以[参考](http://blog.csdn.net/smaller_coder/article/details/52755853) 文章写得很详细。