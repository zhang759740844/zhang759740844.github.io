title: iOS 证书相关
date: 2017/3/4 16:07:12  
categories: iOS
tags:

	- Xcode
---

关于苹果的证书问题搞得焦头烂额，代码没问题，跑不起来这是最气的。所以决定完整地了解一下。

<!--more-->

## 基础

### 密码基础

加密方法可以分为两大类。一类是**对称加密**（private key cryptography），还有一类叫做**非对称加密**（public key cryptography）。前者的加密和解密过程都用同一套密码，后者的加密和解密过程用的是两套密码。

对称加密情况下，密钥只有一把，加密解密都通过其完成。密钥的保存非常重要。

非对称加密时，密钥有两把，一把给别人使用的公钥，一把只有自己知道的私钥。公钥和私钥一一对应，公钥能解开私钥加密的信息，私钥也能解开公钥加密的信息。同时生成公钥和私钥比较容易，但是从公钥推算出私钥是几乎不可能的。

目前，通用的**对称加密算法为AES**（Advanced Encryption Standard），通用的**非对称加密算法为RSA**（ Rivest-Shamir-Adleman）。

在非对称体系中，**公钥用来加密信息（因为只有私钥才能打开），私钥用来数字签名（保证数据完整性的，但它不保证数据加密，不保证数据传输途中无人嗅探窃听）**。

因为任何人都可以生成自己的（公钥，私钥）对，所以为了防止有人散布伪造的公钥骗取信任，就需要一个可靠的第三方机构来生成经过认证的（公钥，私钥）对。

### 非对称加密原理

关于非对称加密，在之前写过的 “http原理” 一文中也有大概的梳理。这边再仔细说下。

需要明确：公钥任何人都能有，私钥只能自己持有，如果别人拿到了你的私钥，那么别人也就成为了你。因此，用公钥加密，私钥解密的信息是绝密的；而用私钥加密，公钥解密的信息，并不能保证安全。所以用私钥加密的信息一般用来保证传输信息的完整性与未被修改。

整个加密解密的流程大概是这样的：

A 有公钥，B 有私钥。A 将用公钥加密的消息发送给 B，那么只有拥有私钥的 B 才能够解密，其他人无法解密出消息明文。

B 要发消息给 A。由于公钥是公开的，那么 B 用私钥加密的消息对所有人其实都是公开的，所以一般不用 B 的私钥加密消息。而只是拿 B 的私钥产生一个“数字签名”。数字签名就是 将 B 即将给 A 发送的消息的 hash 值（通过 MD5、SHA等算法）进行加密。B 发送消息的时候，将消息原文和数字签名一并发出。A 接收到两者后，对原文取 hash，与公钥解密的数字签名中获得的 hash 值比对，如果相同表示消息没有经过别人的修改。

上面保证了 A 的信息能安全的到 B。但是没有解决“如何才能使 B 给 A 发送的消息不被别人窥探？” 很容易想到，只要再来一套公钥私钥，即 B 同时也拥有 A 的公钥，发信息的时候用 A 的公钥加密即可，不过这样处理非常麻烦费时。https 是这样做的：客户端拥有服务端的公钥（即上面所说的 A，先不要管怎么拿到的服务端公钥的以及是如何确定就是服务端公钥而不是其他攻击者的公钥的这两点），服务端拥有其自己的私钥（即 B）。A 在本地生成一段随机字符，作为之后对称加密的密钥，将这个密钥通过公钥加密发给 B，这样 A B 就共同拥有了其他人都不知道的共同密钥，两者之间的通信就安全了。**所以 https 的本质是对称加密，核心是客户端如何将密钥传给服务端，根本问题就是如何能够正确的拿到服务端的公钥，而不被中间人篡改。**

上面说到如何拿到服务端的公钥，在客户端发起请求后，服务端会返回一个证书，这个**证书中就包含着公钥**。客户端如何相信这个证书就是想要请求的服务端的证书而不是攻击者的证书呢？这就需要一个权威的第三方的验证，即"证书中心"（certificate authority，简称CA），为公钥做认证。证书中心**用自己的私钥为服务器的公钥，以及服务端的信息做加密**，连同 CA 信息，保存在新生成的"数字证书"中。客户端最终获取到证书就是经过证书中心加密过的证书。拿到证书后，客户端先到自身根证书列表查看该证书中心是否是本机信任的证书中心（自己计算机的物理安全是前提，如果别人已经可以直接控制你的计算机，修改根证书列表，那什么证书安全也救不了你。）。然后就用根证书里的公钥去解析该证书，验证服务端信息以及公钥是否被修改。

其实道理就是：**你不是不相信发来的公钥的可靠性么。那么就你自己先保存一些信任的第三方的公钥，通过 CA 的公钥去解析获得服务端的公钥。攻击者是没有 CA 认证过 ，或者即使认证了，证书中的信息也是与真正要请求的服务端不同的。因此，这样就能保证其可信性了。**

说完了 https，其实 SSH 也是类似的，服务端传输公钥给客户端给其加密使用，同样的也会遇到中间人攻击的问题。由于 SSH 没有 CA 认证，所以在第一次连接服务器时要慎重，会提示你是否接受这个远程主机的公钥：

```
Are you sure you want to continue connecting (yes/no)? yes
```

如果接受，那么该公钥就会保存在本机中。下次再连接这台主机，系统就会认出它的公钥已经保存在本地了。

https 和 SSH 都接受用户名密码登录，即客户端通过服务器下发的公钥加密后传输给远程主机。SSH 还接受公钥登录，就是本机生成一对公钥和私钥（保存在根目录的 .ssh 文件夹下），将公钥保存在服务器上，有点**类似于上面说的 A 和 B 各自保存自己的公钥和对方的私钥**。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

**在 Apple 开发网站上传包含公钥的 CSR 文件作为换取证书的凭证（Upload CSR file to generate your certificate）（即自己的公钥），有点类似为github账号添加SSH公钥到服务器上进行授权。** 

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

#### 证书分类

iOS证书按功能分为两种：**开发证书（Development）** 和**生产证书（Production）**。

- 开发证书用于开发和调试应用程序，可用于联机调试。无论是 debug 还是 release 用的都是开发证书
- 生产证书用来发布应用程序。

iOS 证书按类型分为两种：**个人证书（Personal）** 和**企业证书（enterprise）**

- 个人证书仅会在安装的时候验证证书的有效性。因此，当你安装完 app 后，即使你删除 developer 账号中的开发证书或者生产证书都是没有关系的。
- 企业证书在任何打开 app 的时候都会验证证书的有效性。因此，无论何时，一定不能删除 developer 账号中的开发证书或者生产证书。否则，用该证书签名的 app 将提示证书错误，将无法打开。

#### 生成证书请求文件 （CSR）

CSR 是 Cerificate Signing Request 的英文缩写，即证书请求文件，也就是**证书申请者在申请数字证书时由 CSP (加密服务提供者)在生成私钥的同时也生成证书请求文件（私钥和 CSR 同时生成）（CSR 其实也就是公钥）**，证书申请者只要把 CSR 文件提交给证书颁发机构后，证书颁发机构使用其根证书私钥签名就生成了证书公钥文件，也就是颁发给用户的证书。通过如下方式申请：

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

点击生成，就**完成了苹果用私钥对你的公钥加密生成证书的过程**。**将生成的证书下载后双击导入钥匙串，自动和之前生成的私钥匹配。**在钥匙串的我的证书中就能看到相应证书。

#### 导出证书

对于团队合作，别人需要你创建的证书，你就要将**苹果下发的证书（用苹果私钥加密的你的公钥）以及相应私钥导出为 .p12 格式的文件**，发给别人使用。注意，**你导出 p12 的时候一定要将证书和私钥一起选中导出**

#### 证书来自某人的 MacBook

不知道你有没有注意过，当你登录到 developer 账号中查看证书的时候，经常会看见如下的情况：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/iOS_cer_1.png?raw=true)

这种带括号某某人的 MacBook 的证书怎么产生的呢？这是因为我们在 Xcode 中登录了开发者账号，如下所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/xcode_account_1.png?raw=true)

登录 AppleId 后，会显示你一个 User 的 Team 一个 Agent 的 Team 里。Xcode 自动帮你在签名的时候生成了一个私钥以及证书。

### Devices

Device是指运行iOS系统用于开发调试App的设备（即苹果设备）。每台Apple设备使用UDID来唯一标识。

设备的UDID可通过iTunes->Summary或者Xcode->Window->Devices获取。

添加设备比较简单，就是输入设备的 UUID，并且给设备取一个别名即可。

### Provisioning Profiles

Provisioning Profile 包含了上述的所有内容：证书，app id，设备。用来将证书和 app id 绑定。

Provisioning Profile 也分为 Development 和 Distribution 两类，有效期同 Certificate 一样。**Development 版本的 ProvisioningProfile 用于开发调试，包括 debug 和 release 版本**，Distribution 版本的 ProvisioningProfile 只用于提交 App Store 审核，其不指定开发测试的 Devices。

在 XcodeTarget->Build Settings->Code Signing->Provisioning Profile 可选择“Automatic”，xcode 会根据该 Target 的“Bundle identifier”选择默认的配置文件及证书：

![provision profile](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_profile.png?raw=true)

**自动签名的情况下，不需要自己设置 Provisioning Profiles**。不过你也可以选择手动设置签名：

![provision profile](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ios_certificate_manual.png?raw=true)

debug 和 release 用的都是开发的配置文件。



更多这方面的信息可以参考[iOS 各种证书/签名详解](http://blog.csdn.net/smaller_coder/article/details/52755853)文章写得很详细。



## iOS app 签名相关

### 目的

对 APP 进行签名的目的很简单，就是**为了验证，你下载的 app 是合法的经过苹果审核的**。

### 简单的签名方式

下面图中有说明：

![签名验证](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verify-certificate.png?raw=true)

这种简单的签名方式是苹果自己生成了私钥和公钥，公钥保存在每一个生产出来的 iOS 设备中。通过苹果的私钥和公钥直接对 app 进行签名和验证。

### 新的问题 

 这种方式面临的问题是所有的 app 都要经过苹果的 app store。但是，比如企业发布啊，开发时的直接调试啊，都不经过 app store。我们面临两个需求：

1. 安装包不需要传到苹果服务器，可以直接安装到手机上。如果你编译一个 APP 到手机前要先传到苹果服务器签名，这显然是不能接受的。**即不能直接用苹果的私钥对 app 进行签名，因为这样必然要将 app 上传**
2. 苹果必须对这里的安装有控制权，包括经过苹果允许才可以这样安装，不能被滥用导致非开发app也能被安装。**即要保证 app 经过了苹果的私钥和公钥的签名和验证过程**



### 两步签名

明白了需求就容易找到思路了。**既然不能将app的签名验证放在云端（苹果），那么就把验证放在本地**。

首先不能用苹果的私钥签名 app，那么很容易想到，我们可以自己创建一对钥匙，用我们自己的私钥对 app 签名，公钥对 app 验证。

为了满足需求二，我们将我们自己创建的公钥上传到苹果的服务器让苹果用其自己的私钥签名（**这就是所谓的开发者证书**），这样就避免了传 app 进行签名的麻烦。当装到手机中后，会将苹果下发的证书以及我们的私钥对 app 的签名导报到设备中（**我们的私钥没有放到设备中，只是将私钥对app 的签名放进去了**）。通过设备中的苹果公钥就能解析出我们自己创建的公钥了。然后用公钥对私钥对app的签名进行解密，验证应用的完整性。**这些操作都是在本地完成的，所以本地安装的时候就不需要联网请求任何东西进行验证了**。

具体流程如下图：



![两步签名](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/2step-verify-certificate.png?raw=true)

1. 在你的 Mac 开发机器生成一对公私钥，这里称为**公钥L**，**私钥L**。L:Local

2. 苹果自己有固定的一对公私钥，跟上面 AppStore 例子一样，私钥在苹果后台，公钥在每个 iOS 设备上。这里称为**公钥A**，**私钥A**。A:Apple

3. 把公钥 L 传到苹果后台，用苹果后台里的私钥 A 去**签名公钥 L**。得到一份数据包含了公钥 L 以及其签名，把这份数据称为证书。

4. 在开发时，编译完一个 APP 后，用本地的私钥 L **对这个 APP 进行签名**，同时把第三步得到的证书一起打包进 APP 里，安装到手机上。

5. 在安装时，iOS 系统取得证书，通过系统内置的公钥 A，去验证证书的数字签名是否正确。

6. 验证证书后**确保了公钥 L 是苹果认证过的**，再用公钥 L 去验证 APP 的签名，这里就间接验证了这个 APP 安装行为是否经过苹果官方允许。

   > 注意，使用苹果的私钥去签名开发者的公钥，开发者的私钥去签名 APP。
   >
   > 想要确保什么的完整性，就用私钥去签名什么

### 更多细节

上面已经是基本的流程了。不过苹果再加了两个限制，一是限制在苹果后台注册过的设备才可以安装，二是限制签名只能针对某一个具体的 APP。除了 设备 ID / AppID，还有其他信息也需要在这里用苹果签名，像这个 APP 里 iCloud / push / 后台运行 等权限苹果都想控制，苹果把这些权限开关统一称为 Entitlements，它也需要通过签名去授权。于是苹果另外搞了个东西，叫 Provisioning Profile，一个 Provisioning Profile 里就包含了证书以及上述提到的所有额外信息，以及所有信息的签名。

流程就变成了下面的样子：

![完整签名流程](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/certificate-all.png?raw=true)



### 签名与证书的关系

我们看到**用私钥对 app 加密叫做签名**，**用私钥对别人的公钥进行加密叫做证书**。

### 苹果的验证机制和 https 验证机制的异同

其实只要涉及到证书的都是为了拿到公钥。https 中服务器拿公钥向 CA 要取签名，苹果的验证中则开发者拿公钥向苹果要取签名。https 中浏览器中有 CA 机构的公钥，苹果的验证中 iOS 设备中有苹果的公钥。

不同的是，https 中，浏览器拿到公钥，是为了加密数据；而 iOS 设备中拿到公钥，是为了验证签名。



### 总结

所以对于上面的各个概念应该有了更深的理解：

1. **证书cer**：内容是公钥或私钥，由其他机构对其签名组成的数据包。
2. **Entitlements**：包含了 App 权限开关列表。
3. **CertificateSigningRequest**：本地公钥。
4. **p12**：本地私钥或者证书或者私钥与证书的二级制形式，可以导入到其他电脑。
5. **PEM：**本地私钥或者证书或者私钥与证书的base64形式，与 p12 类似。
6. **Provisioning Profile**：包含了 证书 / Entitlements 等数据，并由苹果后台私钥签名的数据包。

上面说的是开发时或者是企业的签名验证流程。AppStore 的签名验证方式有些不一样，前面我们说到最简单的签名方式，苹果在后台直接用私钥签名 App 就可以了，实际上苹果确实是这样做的，如果去下载一个 AppStore 的安装包，会发现它里面是没有 embedded.mobileprovision 文件的，也就是它安装和启动的流程是不依赖这个文件，验证流程也就跟上述几种类型不一样了。

那为什么发布 AppStore 的包还是要跟开发版一样搞各种证书和 Provisioning Profile？猜测因为苹果想做统一管理，Provisioning Profile 里包含一些权限控制，AppID 的检验等，苹果不想在上传 AppStore 包时重新用另一种协议做一遍这些验证，就不如统一把这部分放在 Provisioning Profile 里，上传 AppStore 时只要用同样的流程验证这个 Provisioning Profile 是否合法就可以了。

所以 App 上传到 AppStore 后，就跟你的 证书 / Provisioning Profile 都没有关系了，无论他们是否过期或被废除，都不会影响 AppStore 上的安装包。



更多这方面的信息可以[参考bang的文章](http://blog.cnbang.net/tech/3386/) 文章写得很详细。







