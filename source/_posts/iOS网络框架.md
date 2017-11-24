title: iOS 中的网络框架
date: 2017/9/19 10:07:12  
categories: iOS
tags:
	- 架构
---

最近在看巩固网络请求相关的知识。准备把 AFNetworking 以及基于 AFNetworking 的一些封装库看一看，了解一下原理。这一篇将探究一下 AFNetworking 的两个封装库 Casa 的 CTNetworking，以及猿题库的 YTKNetwork 的实现过程。

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

如果这个 API 可以缓存，那么再通过 `serviceType`，`methodName` 以及 `requestParams` 组成的字符串，使用 `CTCache` 这个单例去**查看本地是否有数据**。取出一个自定义的 `CTCachedObject` 类型。这个类型保存两个 `NSData` 对象，一个是缓存的数据 `content`，一个是缓存的时间 `lastUpdateTime`，因为要判断**是否超过了缓存的时间**。如果既有缓存内容，缓存还没有超时，那么拿出来，在主线程中执行成功回调。

不论是从本地还是从 Cache 都使用了**生成了一个返回实例**的封装 `CTURLResponse`，这个封装里有一个标识符 `isCache`，表示非网络请求得到的数据，以此和网络请求数据进行区别。它提供了三个初始化方法，上面两个是网络返回成功和失败的初始化的方法。最后一个 `initWithData:` 就是刚才说的本地数据的初始化方法。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_8.png?raw=true)

如果没有缓存，那么就要执行网络请求了。但是在请求前，还需要**检查一下网络情况**。在上面的配置中心(`CTNetworkingConfigurationManager`，关于配置的都扔在这里)还提供了一个方法调用 AFNetworking  的方法，只要不是明确的无网络 `NotReachable` 就返回 YES。如果无连接，那么执行失败回调， `errorType` 为无网络连接。 

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

将 `CTService` 中的几个属性分为了线上和线下版本。所有实现该协议的类都需要定义这些属性的 get 方法。事实上，我们定义的子类不需要设置 `CTService` 的几个属性。它会自动根据当前环境 `isOnline` 进行选择，`isOnline` 可以统一读取配置信息中的设置。例如：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_12.png?raw=true)

> CTServiceProtocol 定义了几个需要实现的读写方法。CTService 不实现 CTServiceProtocol 协议，但是实现了协议中的读写方法，这样 CTService 的子类就不需要再重复写读写方法了。CTService 的子类要做的就是提供 CTServiceProtocol 中定义的几个属性。相关逻辑就会通过 CTService 中提供的读写方法获取这几个属性值了。

由于可能有多个 service，并且如果每次都创建 Service 太损耗性能，所以就提供了一个工厂 `CTServiceFactory` 用于统一管理保存所有的 service。APIManager 通过 `serviceType` 字符串表示其属于哪个 service，即要创建的 service 类型。因此，我们需要一个 `serviceType` 和 `CTService` 的映射表。这个映射表由一个实现了 `CTServiceFactoryDataSource` 协议的类提供。一般我们将 `AppDelegate` 设置为这个 DataSource，提供 type → Service 的映射。

回到主流程。先到 `CTServiceFactory` 中查看是否已经**存在相应的 Service**。如果不存在就要通过映射表找到对应的 Service 类，创建一个。

得到 Service 后，就可以进行完整 **url 的拼接**。这一操作也是由 `CTService` 完成的。因为不同 Service 可能有不同的拼接规则。

之后**添加额外的参数**。这个参数不是业务级的，也不是接口级的，而是 Service 级的，即所有该 Service 都需要传递的参数，比如 token 之类的。

调用 AFNetworking 提供的方法，传入 url、参数和请求类型，**创建一个 `NSURLRequest` 实例**。

有一些 Service 需要添加自己特有的 header。因此，调用 `CTService` 中的方法，为不同 Service**添加 header**。

Request 本生是没有请求参数这个属性的。但是我们想**把请求参数保存**起来，因为某些情况下，在成功失败回调的时候可能需要用到请求参数。所以我们需要为 `NSURLRequest` 添加一个分类，用于保存请求参数。

现在 Request 生成完毕

#### 执行 Request

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_13.png?raw=true)

执行过程清晰了很多。通过上面创建的 Request，创建一个 `NSURLSessionDataTask` 的实例 `dataTask`。然后通过拿到该 task 的 `taskIdentifier` 作为这个请求的 `requestId`。在 `CTApiProxy` 中保存了一个 `dispatchTable` 字典，用于存储 `requestId` 对应的 `dataTask`。接着执行这个 `dataTask`，再将 `requestId` 一层层传出，`CTAPIBasemanager` 中有一个保存这些 id 的数组 `requestIdList`。这个 `requestId` 的用处在于通过这个 `requestId` 可以随时取消请求。取消的时候通过 `requestIdList` 中的`requestId` 找到 `dispatchTable` 中的 `NSURLSessionDataTask` 实例，然后调用其 `cancel` 方法。

但是要注意，当 API 请求速度非常快的时候，APIManager 并不能确保当前正在进行的请求所对应的 `requestId` 的正确性。所以如果要取消请求的时候，最好 cancelAll，而不是只取消某一个。

### 回调

#### 统一的回调过程

在完成了 `dataTask` 的执行后，会由 `APIProxy` 执行统一的回调过程。回调会传入响应信息 `NSURLResponse *response`，响应体`id responseObject`，以及错误 `NSError *error`。因为请求已经完成，所以移除 `dispatchTable` 中相应的项。如果出错了，那么生成一个出错的返回对象 `CTURLResponse`，如果没有出错，那么生成一个正确的返回对象 `CTURLResponse`，然后将这个 `CTURLResponse` 传入由 `CTAPIBaseManager` 传入的成功失败回调。对于响应信息的日志打印，也是在这里完成。

`CTURLResponse` 没有什么特别的方法，只是添加了一些标识属性，比如前文说到的 `isCache`，以及 `requestId`，`error` 等，还有就是数据。反正你想让回调方法知道的东西都放在这里就可以了。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_14.png?raw=true)

#### 成功回调

成功回调在 APIManager 中进行，除了调用 ViewController 中提供的成功回调外，主要用来保存数据、验证以及拦截。流程如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_15.png?raw=true)

首先是要**保存数据到本地**，这要求该 API 允许从本地获取数据并且响应结果来自于网络，也就是上面所说的 `isCache` 为 NO。这种情况下，用 `NSUserDefaults` 保存网络端传来的响应数据。

前面在网络请求完成时删除了 `dispatchTable` 中的 `requestId` 和 `dataTask` 的键值对。现在要**删除 `requestList` 保存的 `requestId`**。

由于每个 API 都有一个 Manager，所以现在没有必要将数据交给 ViewController 保管了。在 BaseManager 中，定义了一个 `id fetchedRawData` 专门**保存原始的返回数据**。

之后是**验证器**对于返回结果正确性的验证。一般是对于返回状态是否正确、某些特定字段是否为空的判断。这些都由 APIManager 来完成。如果失败，则直接走失败回调，并且设置相应的错误类型。

除了保存到本地就是判断**是否要缓存**。同样的要判断是否能缓存，以及数据是否来此网络。如果可以，调用 `CTCache` 的相应方法更新缓存。

接下来就是在最终**成功回调前的拦截**。拦截分为内部拦截和外部拦截。在 base 中定义了 `beforePerformSuccessWithResponse:` 方法，会调用拦截器的相应方法。如果要实现内部拦截，需要子类重写该方法，并且要调用 super 方法。**成功回调后的拦截**也是相同的道理。

最后要说的就是**成功回调**了。这里要强调一下，并不是所有网络请求都要走最终的成功回调的（至少在本文这个情景下是这样的）。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_16.png?raw=true)

else 的逻辑，不是从本地获取的，那么直接成功回调没有问题。那么从本地获取的数据再分这两种情况，有何用意呢？前面说过，从本地加载的数据成功后会有一次成功回调，之后并没有结束网络请求，而是继续请求新的数据去了。从网络获得数据后又会产生一个成功回调。这里就要把后面这个成功回调屏蔽掉。所以这里先判断是否应该从本地拿数据。如果是，那么就面临着两次回调的情况。本地数据的成功回调是应该执行的，另外本地没有数据时从网络端拿数据也是应该执行的。而本地存在数据，又从网络端拿到的，是不能执行成功回调的。（具体还需要根据实际业务做相应调整的）

#### 失败回调

失败回调较为简单，但是和我们平时写的不一样。由于框架中有很多地方主动调用了失败回调，比如拦截器、验证器发起的失败回调。所以我们需要根据错误类型，先做一些处理。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_17.png?raw=true)

`CTServiceProtocol` 中提供了一个可选方法 `shouldCallBackByFailedOnCallingAPI:`，用来集中处理 Service 层的错误，**比如 token 失效**。我们就可以在 Service 中实现该方法，在其中发出通知。处理完错误后，直接 return，不再继续执行。

所以总的来说，分为三层，**Service 层**，**APIManager 层**和**业务层**，每一层都能进行处理。

### Q&A

#### 链式请求

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
api1.delegate = self;
api2.next = api3;
api3.delegate = self;
[api1 execute];
```

这样做有一个优点，就是有些时候，下一级的网络请求并不依赖于 VC，而是从上一级网络请求直接获得。这样，我们就可以直接设置 `next` 的 `params`，而不需要让 `next` 继续问 VC 要数据了。上面的代码示例里也是这样。首先建立调用链 `api1`→`api2`→`api3`，`api1` 和 `api3` 需要执行 VC 中的回调，就设置其 delegate，`api2` 和 `api3` 都不需要 VC 作为 paramSource，所以就不设置了。

> 这种新建 Command 的好处是，把 delegate，paramSource 做的事放到了 Command 里，不需要在 VC 中处理了。但是这样也带来一个问题就是每个 APIManager 都需要创建一个创建一个与之对应的 Command，导致类目增多。当然，这是不可避免的，毕竟这些 delegate，paramSource 的方法总是要有一个地方放的。
>
> 另外，最好不要将 next 放在 APIManager 中，这样会污染 APIManager。

#### 等待视图在哪实现

等待视图的展示和隐藏在 interceptor 的请求开始前和请求结束后调用。我们应该将加载框设置为一个单例，其中维护一个计数器。当请求开始的时候计数加一，请求结束的时候计数减一。计数器要加锁，因为 API 是异步的。

#### paramsource 是谁？

一般我们的 paramsource 设置为 VC。但是如果不涉及业务上的数据，可以直接设置为 APIManager。

#### 如何分页

将页数保存在 APIManager 中。每次我们要加载下一页的时候使用 `loadNextPage` 方法即可，不需要 VC 管理请求页了。

关于实现，实现方式应该是这样的。首先在 `APIBaseManager` 中定义一个 `pageNo` 属性，设置默认值为 1，表示第一页；一个 `pageSize` 属性，设置默认值为 10。然后还定义 `reloadData` 方法，它的默认实现就是将 `pageNo` 置为 1，然后调用 `loadData`。

再定义一个协议 `PageProtocol`，声明可选的 `pageNo` 和 `pageSize` 属性，再声明一个可选的 `reloadData` 方法，以告诉使用者可以使用。

对于要分页的 API，在 `loadData` 方法中的重组参数方法中，将当前 `pageNo` 传入。在 `APIBaseManager` 中的内部拦截器中，判断当前 APIManager 是否实现了 `PageProtocol` 协议，实现了，就把 `pageNo` 加一。





>简单总结一下，大致分为三步：
>
>每个 API 都有一个 APIManager，其中有 ServiceType，interceptor，delegate，validator，paramSource 等属性。在正真的数据请求前，先对请求参数做一些处理，比如添加 pageNum 等，然后在其中经过一系列的判断，包括 interceptor，validator，是否有本地缓存，是否有 cache 等，涉及到具体的策略。然后进行网络请求。
>
>网络请求通过 APIProxy 完成，它需要 APIManager 提供各种网络请求需要的东西，包括请求参数，请求名，请求类型，ServiceType。它先用这些参数，通过 CTRequestGenerator 创建 Request。创建 Request 的过程是根据 APPDelegate 中定义的对照表，用 ServiceType 在 CTServiceFactory 中找到正真的 CTService 实例，然后获取一些 Service 层面的参数，比如 baseUrl，Service 层的参数等。最后通过 AF 提供的 AFHTTPRequestSerializer 的方法生成 NSURLRequest。
>
>最后 APIProxy 就要发起请求了。还是通过 AF 的方法生成 dataTask，并开始。将 dataTask 和网络请求返回的 requestId 以字典的方式保存起来，又将 requestId 返回给 APIManager 保存。然后等待回调。回调先将 APIProxy 的字典中相应键值对删除，然后删除 APIManager 的数组中的 requestId 删除。



## 猿题库的 YTKNetwork

YTKNetWork 同样是一个 star 数非常高的网络框架，它也是基于离散型 API 完成的。同样的来解析一下 YTK 封装的基本流程。坦白说，CTNetworking 真的很折腾，类目很多，对第一次看的人很不友好。YTKNetwork 则简单了许多，总共也就十来个文件。

### 基本流程

YTK 没有了那么多的 delegate，没有了重组参数，没有了验证器，拦截器，只有关于缓存的讨论。因此，它的主流程也显得简单了许多：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_20.png?raw=true)

首先这个过程基本都是在 `YTKRequest` 中完成的，基本类比于 `APIBaseManager`，所有的 API 都继承于它。第一步判断**是否忽略缓存**，这是各个 API 需要设置的属性。默认为不缓存。

然后判断是**否有下载路径**。也就是各个 API 是否实现了提供下载路径的方法。如果有下载路径，那么就得发送网络请求。

最后是判断**是否能够读取缓存**。先获取各个 API 设置的缓存时间，如果没有设置，那么就得发送网络请求。然后尝试读取缓存数据的元数据。元数据包括缓存的缓存时间，APP 版本号，敏感信息等。这些都要分别判断是否和当前一致。如果没有元数据，或者元数据不符合要求的都得进行网络请求。

如果以上都没有问题，那么执行成功回调，将整个 API 作为参数返回。但是其中任意一个不满足，就会要发送网络请求。发送网络请求前有一个执行准备方法的机会，类似于 interceptor。

### 网络请求

YTK 的网络请求由 `YTKNetworkAgent` 完成，类比于 `ApiProxy`。不过不同的是，YTK 的这个 Agent 做了所有的事，并且包含了文件下载。而 Casa 的 Proxy 则结合 `CTService` 以及 `CTRequestGenerator` 添加了更多的参数配置。YTK 网络请求过程如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_21.png?raw=true)

首先，YTK 允许使用者自己提供 `NSURLRequest` 的实例。所以判断使用者**是否使用了 Request**。如果没有提供，那么需要帮使用者创建一个 Request。

创建的时候首先是对 **detailUrl 的修改**。YTK 提供了一个 `YTKFilterProtocol`，实现了这个协议的类可以对 `YTKRequest` 提供的 detailUrl 进行一定的修改。

之后还可以**修改 baseUrl**。这一过程主要是判断是否使用了 cdn，如果使用了 cdn，那么切换到 cdn 的 baseUrl。这个方法是每个 `YTKRequest` 选择实现的。

**`AFConstructingBlocks`** 是 AFNetwork 中的属性，它一般是在上传文件的时候使用。由  `YTKRequest` 选择完成它的创建。比如下例中将一张图片以 Content-Type 为 `image/jpeg` 的形式上传：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_22.png?raw=true)

准备工作结束，创建 Request 前还需要检查一下该请求**是否是一个下载请求**。还是 `YTKRequest` 提供的可选实现的方法，如果提供了下载路径就表示是一个下载请求。下载的请求涉及到判断路径上文件是否存在，是否可以继续下载等，会通过 AFNetworking 创建一个站稳的 dataTask 类:`NSURLSessionDownloadTask`，所以必须专门创建一个**下载请求的 dataTask**。

不是下载请求的情况下，将上面准备的各种传给 AFNetwork，**生成一个 Request**。之后再将这个 Request 传给 AFNetwork，**得到一个 dataTask**。

之后**设置 dataTask 的优先级**。这个一般也没有太大用处啦。

在执行前，dataTask 肯定是要保存起来的，一来为了方便取消，二来为了到时候能拿到请求的参数等数据。Casa 的实现方式是将 dataTask 保存在 `APIProxy` (对应 `YTKNetworkAgent`)中，以键值分别为 `requestId` 和 dataTask 的字典的形式保存，然后将 `requestId` 以数组的形式又保存在了各个 APIManager(对应 `YTKRequest`) 里。每发送一次请求，数组添加一个 id。VC 可以拿着 id 调用 APIManager 的相关方法来取消请求。

YTK 将 **dataTask 直接保存**在了 `YTKRequest` (对应 APIManager)中。虽然也是通过 `requestId` 和 dataTask 的键值对的方式保存，但是取消并不是以 id 为标识来取消的，而是以 `YTKRequest` 为标识取消的。VC 调用 `YTKNetworkAgent` 的取消方法取消请求。所以 YTK 中每次网络请求，都需要实例化一个 `YTKRequest`。相关过程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_23.png?raw=true)

> 不过就算一个 APIManager 能发多个网络请求了，最后的返回结果还是只有一个并且是会被覆盖的，所以这种保存 requestId 数组的方式并不比每个请求实例化一个 `YTKRequest` 好。

最终通过 dataTask 的 `resume` 就可以开启任务了。

### 回调

#### 统一的回调过程

YTK 的统一的回调过程在 `YTKNetworkAgent` 中完成。主要做了两件事，首先通过 `requestId` 移除保存的 dataTask。其次，将返回的结果设置为 `YTKRequest` 的属性，并验证器有效性。

验证有效性的过程就是在 `YTKRequest` 中提供一个结果的模板，然后比较返回结果和模板的形态是否相同。总的来说就是一个递归验证的过程，就不多赘述了。

#### 成功回调

YTK 的成功回调主要过程就是写入缓存，相比较 CTNetworking 根据需求设置的较为复杂的判断逻辑要直接了许多：就是如果要缓存那么就缓存。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/casa_network_24.png?raw=true)

这个成功回调还是在 `YTKNetworkAgent` 中实现的。在允许缓存并且是从网络获取的情况下写入缓存。这两步也没有太多可以说的。写入缓存也是正常操作。关于创建元数据，前面也说到过了，写缓存的时候一并创建元数据并保存起来。最后的拦截以及成功回调和 CTNetworking 类似。

> 这里成功回调仍然是在 `YTKNetworkAgent` 进行，相比较 CTNetworking 交由 APIManager 处理，这样少了一些定制性。

#### 失败回调

YTK 对于失败的回调基本完全是给到使用者处理了。在失败回调中做的一件事就是对于下载失败的情况，删除未完成的下载项，并将这些数据保存在一个未完成的路径中。

### 批量请求与链式请求

#### 批量请求

批量请求要做的是同时开始多个请求，在最后一个请求完成的时候回调处理方法。YTK 提供了批量请求的封装 `YTKBatchRequest`。虽然也叫 Request，但是它并不继承于 `YTKRequest`。而是保存了 `YTKRequest` 的一个集合。

简单来说，它的实现方式就是初始化的时候传入一个 `YTKRequest` 的集合。在调用 `YTKBatchRequest` 的 `start` 方法开始请求的时候，将每个 `YTKRequest` 的 delegate 都设置为当前的 `YTKBatchRequest`。这样在每一个请求返回的时候，都会走到 `YTKBatchRequest` 的处理方法中。

然后 `YTKBatchRequest` 中为完成的请求计数。当完成的请求小于总请求时，回调方法直接 return。只有当完成的请求等于总请求时，表示全部请求完成，此时执行成功回调。

如果其中有任意一个请求失败了，那么在失败回调中立即为每一个 Request 调用 cancel 方法，取消所有请求。

以上就是批量请求的实现。非常简单。

#### 链式请求

YTK 为链式请求提供了一个 `YTKChainRequest` 类，同样要求在初始化的时候传入一个 `YTKRequest` 的数组。在调用 `YTKChainRequest` 的 `start` 方法开始请求的时候，将第一个 `YTKRequest` 的 delegate 设置为当前的 `YTKChainRequest`。这样在请求返回的时候，都会走到 `YTKChainRequest` 的处理方法中。

除了 `YTKRequest` 的数组，创建的时候还需要传入一个回调闭包的数组。这样，在每个请求完成后，依次执行回调闭包中的方法。

如果其中一个请求失败了，那么执行自定义的失败回调即可。

> 和 CTNetworking 比较，这样的处理方式比较直接，但是不够优雅。

## 总结

CTNetworking 和 YTKNetwork 都是离散型 API 的实现。不过我认为 CTNetworking 的实现方式更为优雅。以协议的方式明确职责，引入 delegate，validator，interceptor，让定制化程度更高，并且减轻了 VC 的负担。引入 CTService 让多服务接入验证变得简单。但是反过来说，这也一定程度上提高了学习成本。并且 CTNetworing 并不支持上传与下载文件。其缓存策略也需要使用者根据实际需求做一定的改写。相比较而言，YTKNetwork 的功能就更纯粹了一些，专注于网络请求，在缓存与下载方面做了充足的处理。

  







 









