title: 读 Casa 应用架构的一些笔记
date: 2017/5/13 10:07:12  
categories: iOS
tags:
	- 架构
---

读了 Casa 的一系列架构的文章 [Archives for Casa Taloyum](https://casatwy.com/archives.html)，自己再总结消化一下。

<!--more-->



## 开篇

### 什么样才是好架构

> 代码整齐，分类明确，没有 common，没有 core

不要让一个类或者一个模块做两种不同的事情（单一职责原则）。一方面不适合未来拓展，另一方面会造成分类困难。

不要搞 Common，Core 这些东西。迭代多了会越来越大。如果有什么东西特别小，索性单独开辟一个模块就行了。原因如下：

1. Common 里容易形成错综复杂的小模块依赖，会纵容工程师不注意依赖的管理，以至于未来想拆分的时候，会很困难。
2. Common 与细粒度模块的设计思想背道而驰，属于偷懒的手段。
3. 每次迭代都会把不好分类的东西放在 Common，给维护 Common 的人带来大的工作量，容易出错。


> 很少文档就能让业务方上手

尽可能将 API 命名的可读性强一些。比如：

1. 不要直接返回 `id` 或者传入 `id`，实在不行，用 `id<protocol>` 也比 `id` 好。
2. 要告知也无妨传什么。比如要传 `Image`，就要写上 `ofImage` ，要传位置，既要写上 `IndexPath` ，而不是 `position` 这种笼统的命名。
3. 没有必要传 `delegate`，`delegate` 可以作为属性赋值。


> 没有横向依赖，万不得已不要跨层访问

横向依赖就是在某一分层中的各模块间的相互依赖的描述。在架构图上如果画出依赖关系，就可以看到他们是横着的，所以叫横向依赖。纵向依赖就是上层依赖下层。

跨层访问是指数据流向了跟自己没有对接关系的模块。跨层访问同样增加耦合度，当某一层需要整体替换的时候，牵涉面就会很大。

> 接口少，接口参数少

越少的接口和越少的参数，就能降低业务方的使用成本。需要在满足条件的前提下，减少接口和参数。

要达到一个功能的最少传参，多的参数没有必要性，还容易产生耦合。



### Q&A

#### 关于依赖层级

依赖关系都是上层依赖下层，因为下层为上层提供服务。

#### 关于如何 import

不要使用 pch，也最好不要专门添加一个头文件 import 所有要用到的类。在你的类中，需要使用到哪个就 import 哪个。

#### 关于拆分pod

category 理想的拆分是按照被 category 的对象拆分成 pod。string 就 string 一个 pod，uiview 就 uiview 一个 pod，然后用刀哪个就引用哪个 pod，pod 多一点没有问题。如果单个 pod 很大，还可以根据业务继续拆。

#### 关于管理图片

可以通过 cocoapods 封装组件，将每个 SDK 及其对应的再封装的代码放到同一个 repo 里去。也可以通过 cocoapods 专门管理一个图片 repo，然后根据模块分文件夹来管理，每次添加图片的时候都去看一下这个 repo 里是不是已经有这个图片了。



## View 层组织和调用方案

### 代码结构的规定

代码需要遵循一定的规范：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_1.png?raw=true)

> 所有属性都使用 getter 方法里

不要在 `viewDidLoad` 里面初始化 view。在 `viewDidLoad` 做 `addSubview` 以及设置布局，初始化的操作都在 getter 方法中。

确实我也觉得初始化都写在 `viewDidLoad` 里很丑。所以写在读取方法里其实也挺好。不要忘了如果写在读取方法里，要有一个判断非空返回的操作，否则会重复创建。

> getter 和 setter 方法最后面

存取方法会有很多，所以最好写在最后面。

> delegate 要写上对应的 protocol 名字，delegate 方法要写在一块区域

老老实实的写成 `#pragma mark - xxxxDelegate`。

> event response 写在一个区域

所有的 button，gestureRecognizer 的响应事件写在一个区域里面。

> 减少 private methods

所有方法，如果是那种小功能如日期转换，图片裁剪，最好把它写成一个 category，有利于复用。



### 胖瘦 model

瘦 model 专门表达数据，存储、处理都交由外部处理。瘦 model 的目的是，尽可能编写细粒度 model，然后配套各种 helper 对弱业务做抽象，强业务依旧交给 Controller。

胖 model 包含了部分弱业务逻辑。胖 model 的目的是，Controller 从胖 model 拿到数据后，不用额外操作或者很少的操作，就能够将数据直接应用在 View 上。胖 model 相对难以移植一些。

我倾向于瘦 model + helper 的形式，这样更加抽象一些。



### 跨业务页面调用

跨业务指的是 App 中存在 A 业务和 B 业务，B 业务有可能要展示 A 业务的某个页面，A 业务也可能要展示 B 业务的某个页面。如果直接 import 会出现以下问题：

1. 如果要新开一个业务 C，且 C 依赖于业务 B，那么就需要连 A 也塞入开发环境中。
2. 如果业务中的某个页面修改了，比如改名，那么除了该业务中的其他页面的逻辑要修改，其他业务与其相关的逻辑也要修改。

因此，我们需要使用一个 Mediator 将业务解耦。具体如何实现的，参考后面的组件化。



### Q&A

#### 关于 Notification 在哪设置

Notification 最好在 `viewDidAppear` 中注册，在 `viewDidDisappear` 中移除，即在页面显示的时候接收通知。实在需要页面不显示的时候也能接到通知的话就只能在 `init` 和 `dealloc` 中做添加删除了（`dealloc` 中 remove 不会造成内存泄漏，Notification 中的 Observer 不是 strong，所以不会让引用+1，但是必须 remove 不然会产生野指针导致崩溃）由于 `viewDidAppear` 和 `viewDidDisappear` 不是成对出现的，需要在 `viewDidAppear` 注册通知前先移除一下该通知。

#### 关于 BaseViewController 和 AOP

我觉得用 aop 代替继承还是要看使用范围。如果像埋点这样的所有情况都要走埋点接口的就适合 aop，而 ViewController 这种自己的要还是要写个 base 重写 `viewDidLoad` 的。比如设置背景色，如果用 aop，那么系统的各种弹窗也会同样设置了背景色。这就需要判断是不是系统的，就还要在所有的基础上做减法，就感觉很蠢。

当然，base 绝对不能乱用，我觉得设置设置背景色，代理类应该是没有问题的。base 还有一个好处是能设置一个代码规范，告诉业务开发者，哪些东西要写在哪个方法里。

####  关于 cell 处理事件

场景：点击一个 cell 要跳转另一个 VC，那么是直接在 cell 里跳转，还是先通知所在 VC，让 VC 跳转？

应该设置一个 delegate 为当前 VC，然后通知 delegate 跳转。cell 这种层次的 View 不应该参合在页面调度这种 VC 才应该做的事情上。



## 网络层设计方案











