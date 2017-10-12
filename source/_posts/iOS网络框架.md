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

其中包括成功与失败的着陆点 `delegate`，提供参数的 `paramSource`，验证参数的 `validator`，以及拦截器 `interceptor`。它们都实现了各自协议。一般来说 `delegate` 是调用的 ViewController，`paramSource` 也是调用的 `ViewController`，`validator` 是 APIManager 自身，`interceptor` 是调用的 ViewController，`validator` 和 `interceptor` 都不是必须的。这些属性需要在初始化的时候设置好。

可以看到，其中还有一个实现 `CTAPIManager` 协议的 `child` 属性，所有派生的 APIManager 都必须实现 `CTAPIManager` 协议。网络请求所需的 API 名，服务类型，请求类型以及是否缓存 (是否缓存其实没必要作为必须的方法吧，默认不缓存，要缓存的子类实现就好了啊) 都需要通过这个接口中返回：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_3.png?raw=true)

所以现在的继承关系应该是：APIManager 继承于 `CTAPIBaseManager`，并且实现协议 `CTAPIManager`。**基类本生并不实现协议**。

那么为什么基类不直接提供协议中的方法，而是要以协议的方式获取呢？**协议的特点就是提供接口规范**。对于这些必须实现的方法，确实是可以通过在基类中抛出异常来强制子类重写，达到目的。但是这有两个缺点，一个是不直观，调用者不能立即知道需要重写哪些方法。另一个缺点是，可能有些方法忘记了重写，这样只有在运行的时候才会抛出异常，但是使用协议，如果没有完全实现必须的方法，编译都无法通过。

因此，在初始化的时候要先判断派生类是否实现了 `CTAPIManager`，没有实现的直接抛出异常。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_7.png?raw=true)

### 基本流程

在创建了 APIManager 的实例后，调用其统一的方法 `loadData` 来开始网络请求。基本流程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_1.png?raw=true)

首先是**获取参数**。一般情况下，我们是在网络请求的时候将参数一并传过去，而这里需要你将实现了  `CTAPIManagerParamSource` 协议的类作为数据源，实现其获取数据的方法，通过 `manager` 的不同来区分不同 API：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_4.png?raw=true)

**重组参数** `reformParams:`，并不是一个必须的方法，默认返回原参数。子类可以重写该方法在调用 API 之前额外添加一些与业务逻辑无关的参数，比如 pageNumber 和 pageSize 之类的。

**拦截器**不是必须的，所以调用拦截器方法前，先要判断一下是否有拦截器。拦截器是和业务逻辑有关的，所以一般是 ViewController，其需要实现 `CTAPIManagerInterceptor` 协议。这个协议中包含了很多拦截方法，包括调用 API 之前之后，请求成功之前之后，请求失败之前之后的回调方法。当前调用的是 `manager:shouldCallAPIWithParams:` 方法，用来检测此 `manager` 是否能以该  `params` 调用网络请求。如果允许调用 API 则进入下一步，如果不允许，直接结束请求(你也可以在不允许的时候设置走到失败回调)。拦截器一般用来展示和隐藏菊花~

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_5.png?raw=true)

接下来是**验证器**，用来验证请求数据以及返回数据的合法性。验证器是和业务无关的，所以一般交给 APIManager 就行了。在 API 调用前，验证器一般用来在 API 调用前验证参数是否符合规则，比如手机号啊，邮箱啊之类，避免无效的 API 请求，加快响应速度。如果验证失败了，直接走失败回调，将`CTAPIBaseManager` 中的 `CTAPIManagerErrorType errorType` 属性设置为 `CTAPIManagerErrorTypeParamsError` 表示参数错误。

 ![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_6.png?raw=true)

接下来询问子类 APIManager 是否要**从本地获取数据**。如果可以从本地获取，那么从 UserDefault 中取出数据。如果有数据，那么在主线程中执行成功回调，如果没有数据，那么设置 `isNativeDataEmpty` 标识位 YES，这个标识位后面会用到。

注意到，这里即使从本地加载了数据，也不会直接 return 而是要继续进行网络请求的。这里的逻辑是可能存在一些 API 获取到数据后不会立即展示，而是保存起来，展示上次保存的数据。比如首屏广告图片。

接下来询问**是否可以缓存**，如果子类没有实现这个方法，那么从 `CTNetworkingConfigurationManager` 这个配置中心获取。（这里不太对，因为 `shouldCache` 是 `APIManager` 协议中定义的必须实现的方法方法，所以不会存在从配置中心读取这一步。其实最好还是直接设置成可选方法，在 base 中提供个默认实现。）

如果这个 API 可以缓存，那么再通过 `serviceType`，`methodName` 以及 `requestParams` 组成的字符串，去**查看本地是否有数据**。取出一个自定义的 `CTCachedObject` 类型。这个类型保存两个东西，一个是缓存的数据 `content`，一个是缓存的时间 `lastUpdateTime`，因为要判断**是否超过了缓存的时间**。如果既有缓存内容，缓存还没有超时，那么拿出来，在主线程中执行成功回调。

不论是从本地还是从 Cache 都使用了**生成了一个返回实例**的封装 `CTURLResponse`，这个封装里有一个标识符 `isCache`，表示非网络请求得到的数据，以此和网络请求数据进行区别。它提供了三个初始化方法，上面两个是网络返回成功和失败的初始化的方法。最后一个 `initWithData:` 就是刚才说的本地数据的初始化方法。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_8.png?raw=true)

如果没有缓存，那么就要执行网络请求了。但是在请求前，还需要**检查一下网络情况**。在上面的配置中心还提供了一个方法调用 AFNetworking  的方法，只要不是明确的无网络 `NotReachable` 就返回 YES。如果无连接，那么执行失败回调， `errorType` 为无网络连接。 

最后，发送网络请求前，将 APIManager 中的 `isLoading` 标志位置为 YES，表示正在进行网络请求。这个的作用是在某些 API 中可以防止重复发送请求。比如翻页，就可以判断 `isLoading` 的状态来执行不同的请求策略。

### 网络请求

网络请求大致分为两步，第一步是生成一个 `NSURLRequest`，第二步就是执行。整个过程在 `CTApiProxy` 中进行。

#### 生成 Request

生成 Request 的主要过程在 `CTRequestGenerator` 中完成。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_9.png?raw=true)



在开始前，我们先要知道 `CTService` 是个什么东西。来看一下它的构成：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_10.png?raw=true)

主要定义了公钥、私钥、根 Url 以及 API 版本。根据这个 `CTService` 可以派生出各种 service，每一个 service 都有自己的 `apiBaseUrl`。所以 `CTService` 的设计是用于描述第三方API的相关配置和信息的。如果没有第三方 API，只要创建一个自身的 Service 就行了。

另外我们看到，同 `BaseAPIManager` 一样，`CTService` 也有一个 `child`。它实现了 `CTServiceProtocol` 协议：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_11.png?raw=true)

将 `CTService` 中的几个属性分为了线上和线下版本。所有实现该协议的类都需要定义这些属性的 get 方法。事实上，我们定义的子类不需要设置 `CTService` 的几个属性。它会自动根据当前环境 `isOnline` 进行选择，例如：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_12.png?raw=true)

`isOnline` 可以统一读取配置信息中的设置。

提供了一个工厂 `CTServiceFactory` 用于统一管理所有的 service。APIManager 中的`serviceType` 字符串就是用来描述其属于哪个 service 的。`serviceType` 字符串和具体的 `CTService` 的映射关系字典由一个实现了 `CTServiceFactoryDataSource` 协议的类提供。一般将这个字典放在 `AppDelegate` 中。

回到主流程。先到 `CTServiceFactory` 中查看是否已经**存在相应的 Service**。如果不存在就要通过映射表找到对应的 Service 类，创建一个。

得到 Service 后，就可以进行完整 **url 的拼接**。这一操作也是由 `CTService` 完成的。因为不同 Service 可能有不同的拼接规则。

之后**添加额外的参数**。这个参数不是业务级的，也不是接口级的，而是 Service 级的，即所有该 Service 都需要传递的参数，比如 token 之类的。

调用 AFNetworking 提供的方法，传入 url、参数和请求类型，**创建一个 request**。

有一些 Service 需要添加自己特有的 header。因此，调用 `CTService` 中的方法，为不同 Service**添加 header**。

Request 本生是没有请求参数这个属性的。但是我们想把请求参数保存起来，因为某些情况下，在成功失败回调的时候可能需要用到请求参数。所以我们需要为 `NSURLRequest` 添加一个分类，用于保存请求参数。

现在 Request 生成完毕

#### 执行 Request

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_13.png?raw=true)

执行过程清晰了很多。通过上面创建的 Request，创建一个 `NSURLSessionDataTask` 的实例 `dataTask`。然后通过拿到该 task 的 `taskIdentifier` 作为这个请求的 `requestId`。在 `CTApiProxy` 中保存了一个 `dispatchTable` 字典，用于存储 `requestId` 对应的 `dataTask`。接着执行这个 `dataTask`，再将 `requestId` 一层层传出，`CTAPIBasemanager` 中有一个保存这些 id 的数组 `requestIdList`。这个 `requestId` 的用处在于通过这个 `requestId` 可以随时取消请求。但是要注意，当 API 请求速度非常快的时候，APIManager 并不能确保当前正在进行的请求所对应的 `requestId` 的正确性。所以如果要取消请求的时候，最好 cancelAll，而不是只取消某一个。

在完成了 `dataTask` 的执行后，执行回调。回调会传入响应信息 `NSURLResponse *response`，响应体`id responseObject`，以及错误 `NSError *error`。因为请求已经完成，所以移除 `dispatchTable` 中相应的项。如果出错了，那么生成一个出错的返回对象 `CTURLResponse`，如果没有出错，那么生成一个正确的返回对象 `CTURLResponse`，然后各自执行自开始传入的成功失败回调。对于响应信息的日志打印，也是在这里完成。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_14.png?raw=true)

### 回调

回调的处理都是在 APIManager 中完成。

#### 成功回调

成功回调除了调用 ViewController 中提供的成功回调外，主要用来保存数据、验证以及拦截。流程如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_15.png?raw=true)

首先是要**保存数据到本地**，这要求该 API 允许从本地获取数据并且响应结果来自于网络，也就是上面所说的 `isCache` 为 NO。这种情况下，用 `NSUserDefaults` 保存网络端传来的响应数据。

前面在网络请求完成时删除了 `dispatchTable` 中的 `requestId` 和 `dataTask` 的键值对。现在要**删除 `requestList` 保存的 `requestId`**。

由于每个 API 都有一个 Manager，所以现在没有必要将数据交给 ViewController 保管了。在 BaseManager 中，定义了一个 `id fetchedRawData` 专门**保存原始的返回数据**。

之后是**验证器**对于返回结果正确性的验证。一般是对于返回状态是否正确、某些特定字段是否为空的判断。这些都由 APIManager 来完成。如果失败，则直接走失败回调，并且设置相应的错误类型。

除了保存到本地就是判断**是否要缓存**。同样的要判断是否能缓存，以及数据是否来此网络。如果可以，调用 `CTCache` 的相应方法更新缓存。

接下来就是在最终**成功回调前的拦截**。拦截分为内部拦截和外部拦截。在 base 中定义了 `beforePerformSuccessWithResponse:` 方法，会调用拦截器的相应方法。如果要实现内部拦截，需要子类重写该方法，并且要调用 super 方法。**成功回调后的拦截**也是相同的道理。

最后要说的就是**成功回调**了。并不是所有请求都要走最终的成功回调的。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_16.png?raw=true)

else 的逻辑，不是从本地获取的那么直接成功回调没有问题。那么从本地获取的数据再分这两种情况回调是什么意思呢？前面说过，从本地加载的数据成功后会有一次成功回调，之后并没有结束网络请求，而是继续请求新的数据去了。从网络获得数据后又会产生一个成功回调。这里就要把后面这个成功回调屏蔽掉。所以这里先判断是否应该从本地拿数据。如果是，那么就面临着两次回调的情况。本地数据的成功回调是应该执行的，另外本地没有数据时从网络端拿数据也是应该执行的。而本地存在数据，又从网络端拿到的，是不能执行成功回调的。

#### 失败回调

失败回调较为简单，但是和我们平时写的不一样。由于框架中有很多地方主动调用了失败回调，比如拦截器、验证器发起的失败回调。所以我们需要根据错误类型，先做一些处理。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_17.png?raw=true)

`CTServiceProtocol` 中提供了一个可选方法 `shouldCallBackByFailedOnCallingAPI:`，用来集中处理 Service 层的错误，比如 token 失效。我们就可以在 Service 中实现该方法，在其中发出通知。处理完错误后，直接 return，不再继续执行。所以总的来说，分为三层，Service 层，APIManager 层和业务层，每一层都能进行处理。

### 博客中的问答

#### 如何处理请求链

如何处理链式请求？一般我们都是在成功回调后拿到数据，然后再进行下一个请求。如果存在多个请求，那么这个成功回调会嵌套的非常长。进行的改进就是争取将各个请求从回调中取出。我们可以将每一个回调放在一个专门的类中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_18.png?raw=true)

如图所示，为每一个请求链上的 API 创建一个 Command 类。这个 Command 类是对 APIManager 的一层封装，ViewController 中不再直接调用 APIManager 中的 `loadData` 方法，而是调用 Command 中提供的 `execute`(名字叫什么不重要)表示我发起了一个请求链。将 Command 设置为 APIManager 的 paramSource，delegate。APIManager 先执行 Command 的回调。Command 中提供了一个同为 Command 类型的 `next` 属性，这就是请求链的下一级。在 Command 的回调中，判断 `next` 是否为空。当存在时，如果下一级要依赖上一级返回的结果作为参数，就将下一级的 paramSource 设置为上一级，然后通过 `[self.next execute]` 执行。如果不存在，那么调用 delegate（一般是 ViewController）的完成回调，表示请求链结束。

使用的时候为每一个 Command 设置 `next` 即可：

```objc
// ViewController 中
API1Command *api1 = [[API1Command alloc] init];
API2Command *api2 = [[API2Command alloc] init];
API3Command *api3 = [[API3Command alloc] init];
api1.next = api2;
api1.paramSource = self;
api2.next = api3;
api3.delegate = self;
[api1 execute];
```

这只是一个思路，具体还需要自己按照需求做出一定的调整。





 









