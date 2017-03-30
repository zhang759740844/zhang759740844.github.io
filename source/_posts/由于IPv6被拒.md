title: 因为IPv6被拒之后
date: 2017/3/31 10:07:12  	
categories: iOS
tags:

	- 学习笔记
---

最近的两个 app 相继被拒。被拒的原因都是由于 IPv6，导致网络无法连接。一般来说需要更新网络库。但是也说不定是审核的问题啊。所以可以先搭建一个本地的 IPv6 的环境，测试下是否是自己的问题。

<!--more-->

### 搭建步骤

可以通过 mac 构建一个 IPv6 的环境，然后让手机连接上 mac。但是这就需要一个转接口，因为需要 mac 直接连接到以太网。如果没有转接口，那么就可以参照下面的方法需要两台 iphone（一部提供网络，一部用于测试） 以及一根数据线。

第一步，将用于提供网络的机器通过数据线连接上mac。

第二步，关闭机器的 wifi 和 4G，打开机器的个人热点并且选择仅 USB 如下图所示：

![打开个人热点](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/selfWifi.png?raw=true)

第三步，打开系统偏好设置,按住option(alt)键（一定要按）点击共享：

![打开共享](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ipv6_share.png?raw=true)

第四步，创建 NAT64 网络

![打开个人热点](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ipv6_createWifi.png?raw=true)

第五步，关闭 mac 当前连接的wifi，等待一会就会显示如下图片所示的图标：

![连接个人热点](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ipv6_openWifi.png?raw=true)

第六步，测试用的手机连接 wifi

![打开个人热点](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ipv6_connectWifi.png?raw=true)

如果连接正确，如果显示如上图的 DNS，那么就表示 IPv6 的环境已经搭建好了。可以测试是否是自己代码的问题，还是苹果测试的问题了。



### 依旧被拒

如果依旧被拒那么可以拍一个视频给审核人员[详情参考这个链接,有详细例子如何拍视频,点击查看](http://www.cocoachina.com/bbs/read.php?tid-1684531.html),[如何录制视频的样例,有几位按照这个录制顺利通过审核](http://v.qq.com/x/page/i03059ch09b.html)

**问题:如何拍视频啊?** 答:拿个安卓或者iOS手机拍摄.个人觉得不应该是录制屏幕,录制屏幕不能很好的反映出你适配ipv6的过程 
**问题:怎么拍** 答:先拍你搭建环境的过程,ipv6环境搭建好了,wifi有箭头吧,ipv6 的DNS是:隔开的,iPv4 的DNS 是隔开的,手机正确连接电脑wifi的过程也需要拍摄的,这些标志你搭建ipv6环境搭建成功的画面都需要 拍,在拍的时候把自己的app所有界面(都可以加载数据)运行良好的状态拍一下 
**问题:拍好的视频怎么传给苹果审核人员?** 答:拍好的视频传到youtwobe,(不推荐传到优酷,万一美国的审核人员没有耐心等待你的视频加载,又给你打回来 了,美国访问中国的网速会比中国访问中国的网速要慢) 如果你重新提交新版本,就把链接贴在备注的描述 下,平时在这个描述里写这个app.如果你的app 你觉得没任何问题,不想再上传ipa包,登录苹果开发者账号 找到苹果拒绝的描述,这个描述是可以回复的(Reply),在这个Reply里面贴上你的视频链接,写上大致意思 是:"我真的适配了ipv6,我把适配和测试过程都排了,麻烦你再审一遍"的话,说话一定要诚恳,礼貌,说话一定要诚恳,礼貌,说话一定要诚恳,礼貌(重要事说三遍),上面的这些做了还是被拒绝怎么办,措施1试过,措施2也试过,还是不管用,我只能建议你继续提交了,stakeoverflow上面有人说,自己测试了都是没问题,怎么苹果还是拒绝,苹果的工程师告诉他你就继续提交吧,这是审核人员的bug,其实这种情况国内的开发者(包括笔者)也遇到过,你明明提供了app的测试账号,他说你没提供,你回复下就好了!他们人工审核也会有失误的时候,只是这个失误被你碰到了!这就是为什么有部分网友啥都没修改,只是在拒绝的描述哪里reply 一下就通过了.



参考文章：

[iOS-用手机网络测试Ipv6](http://www.jianshu.com/p/6c7a155fc372)

[ipv6审核被拒绝的解决方案](https://github.com/wg689/Solve-App-Store-Review-Problem/blob/master/ipv6.md#q三实在搞不定ipv6怎么办对ipv6无计可适的时候可以考虑)