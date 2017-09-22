title: iOS 中的网络框架
date: 2017/9/19 10:07:12  
categories: iOS
tags:
	- 架构
---

学习一下一些网络封装框架的设计

<!--more-->

## Casa 的 CTNetworking

Casa 关于网络架构的博客中主要讲了如何渔，没有分析他的鱼是什么样的。刚看的时候，我是一头雾水，相关的类目太多了，有些类还不太能够理解意图。进过一段时间的研究，现在明白了各个环节的作用，做一个整理。

### 初始化

由于采用的是离散型的 API，每一个 API 都对应一个 APIManager，所以进行网络请求的时候，要分别初始化各自需要的 APIManager。初始化 APIManager 的时候，要设置一些 APIBaseManager 中的属性，如下图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_2.png?raw=true)

其中包括成功与失败的着陆点 `delegate`，提供参数的 `paramSource`，验证参数的 `validator`，以及拦截器 `interceptor`。它们都实现了各自协议。一般来说 `delegate` 是调用的 ViewController，`paramSource` 也是调用的 `ViewController`，`validator` 是 APIManager 自身，`interceptor` 是调用的 ViewController，`validator` 和 `interceptor` 都不是必须的。

可以看到，其中还有一个实现 `CTAPIManager` 协议的 `child` 属性，所有派生的 APIManager 都必须实现 `CTAPIManager` 协议。网络请求所需的 API 名，请求类型等参数都需要通过这个接口中返回：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_3.png?raw=true)

所以现在的继承关系应该是：APIManager 继承于 `CTAPIBaseManager`，并且实现协议 `CTAPIManager`。基类本生并不实现协议，但是协议的那几个可选方法都已经在基类中定义实现过了，仍然在协议中声明的原因是让调用者更清楚的意识到自己能调用哪些方法。

(我觉得，Casa 设置 child 这个属性的本意是为了让 APIManager 不继承于 `CTAPIBaseManager`，通过组合的方式代替继承，所以这里的 `child` 也被设置成了 `NSObject` 类型。关于继承还是组合，我认为：要么就有 `child`，那么 APIManager 就不要继承于 `CTAPIBaseManager` ，以组合方式实现；要么就不要使用 `child`，直接派生于 `CTAPIBaseManager` ，以继承方式实现。但是事实上 casa 的 demo 中的 APIManager 还是继承于 `CTAPIBaseManager` 的，所以我感觉，没有必要专门定义这么一个 `child` 属性，况且就算以组合的方式实现了，也没有达到什么好处。)



### 基本流程

在创建了 APIManager 的实例后，调用其统一的方法 `loadData` 来开始网络请求。基本流程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_1.png?raw=true)

首先是获取请求的参数。一般情况下，我们是在网络请求的时候将参数一并传过去，而这里需要你将实现了  `CTAPIManagerParamSource` 协议的类作为数据源，实现其获取数据的方法，通过 `manager` 的不同来区分不同 API：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_4.png?raw=true)

重组参数 `reformParams:` 本身就是基类 `CTAPIBaseManager` 以及协议 `CTAPIManager`中的方法，子类可以重写该方法在调用 API 之前额外添加一些参数，比如 pageNumber 和 pageSize 之类的，默认返回原参数。这些可以不在 ViewController 中管理，而放在 APIManager 中，减轻 ViewController 的负担。（token 之类的要不要也在这里添加呢？）

拦截器不是必须的，所以调用拦截器方法前，先要判断一下是否有拦截器。拦截器是和业务逻辑有关的，所以一般是 ViewController，其需要实现 `CTAPIManagerInterceptor` 协议。这个协议中包含了很多拦截方法，包括调用 API 之前之后，请求成功之前之后，请求失败之前之后的回调方法。当前调用的是 `manager:shouldCallAPIWithParams:` 方法，用来检测此 `manager` 是否能以该  `params` 调用网络请求。如果允许调用 API 则进入下一步，如果不允许，则设置相应的 `errorType`，然后执行失败回调（casa 的源代码其实是在不允许的时候，直接返回什么也没有做的，我觉得还是应该有相应失败回调的）。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_5.png?raw=true)

接下来是验证器，用来验证数据的合法性。验证器是和业务无关的，所以一般交给 APIManager 就行了。在 API 调用前，验证器一般用来在 API 调用前验证参数是否符合规则，比如手机号啊，邮箱啊之类，避免无效的 API 请求。在 API 返回之后，验证器一般用来判断返回的某些数据是否为空（判断返回状态应该是在基类中去完成的，到这一步的时候是已经解析得到具体的数据了），将原本 ViewController 要做的事移到了 APIManager 中去。

 ![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_6.png?raw=true)

允许从本地获取数据 `shouldLoadFromNative` 方法是基类 `CTAPIBaseManager` 以及协议 `CTAPIManager`中的方法，子类重写以判断是否能从本地获取数据。默认不从本地获取。当可以从本地加载数据的时候，













