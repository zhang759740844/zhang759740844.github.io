title: WKWebView 使用（二）—— 常见问题
date: 2018/8/31 14:07:12  
categories: iOS
tags: 

- 学习笔记

------

记录一下 WKWebView 相关使用

<!--more-->

## Cookie 管理

### loadRequest 无 Cookie

WKWebView 不会在请求的时候自动到 `NSHTTPCookieStorage` 中获取 cookie(UIWebView 可以)。所以需要我们在  `loadRequest` 前，手动从 `NSHTTPCookieStorage` 中拿到 Cookie，并将处理好的 Cookie String在 request header 中设置 Cookie, 解决**首个请求** Cookie 带不上的问题：

```objc
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://www.baidu.com"]];
// 根据 Request 的 URL，获取相应的 cookie
NSArray *availableCookie = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL];
// 重新创建一个可变数组
NSMutableArray *cookieMarr = [NSMutableArray arrayWithArray:availableCookie];
//删除过期的cookie
for (int i = 0; i < cookieMarr.count; i++) {
    NSHTTPCookie *cookie = [cookieMarr objectAtIndex:i];
    if (!cookie.expiresDate) {
        continue;
    }
    /// cookie 有 expiresDate，超过的就 remove 掉
    if ([cookie.expiresDate compare:self.currentTime]) {
        [cookieMarr removeObject:cookie];
        i--;
    }
}
// 把 cookie 的 array 转为 string 类型
for (NSHTTPCookie *cookie in cookieArr) {
    if ([cookie.name rangeOfString:@"'"].location != NSNotFound) {
        continue;
    }
    
    if (![validDomain hasSuffix:cookie.domain] && ![cookie.domain hasSuffix:validDomain]) {
        continue;
    }
    
    NSString *value = [NSString stringWithFormat:@"%@=%@", cookie.name, cookie.value];
    [marr addObject:value];
}
NSString *cookie = [marr componentsJoinedByString:@";"];

// 设置 request 的 cookie
[request setValue:cookie forHTTPHeaderField:@"Cookie"];
[self.webView loadRequest:request];
```

重定向相关：

当服务器发生重定向的时候，此时第一次在 RequestHeader 中写入的 Cookie 会丢失，还需要重新对重定向的 NSURLRequest 进行 RequestHeader 的 Cookie 处理 ，简单的说就是在 `webView:decidePolicyForNavigationAction:decisionHandler:` 的时候，判断此时 Request 是否有你要的 Cookie 没有就Cancel掉，修改Request 重新发起。

> iOS 11 中包含 `WKHTTPCookieStore` 相关的 API，可以解决 WKWebView Cookie 的问题。

### JS 在执行的时候使用 document.cookie 读取不到 cookie

当生成 Request 后，页面加载之前，给 WKWebView 注入脚本，使js 端的 cookie 和 `NSHTTPCookieStorage` 同步：

```objc
-(void)syncClientCookieScripts:(NSMutableURLRequest *)request{
    if (!request.URL) {
        return;
    }
    NSArray *availableCookie = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL];
    NSMutableArray *filterCookie = [[NSMutableArray alloc]init];
   
    for (NSHTTPCookie * cookie in availableCookie) {
        if (self.syncCookieMode) {
            //httponly需求不得写入js cookie
            if (!cookie.HTTPOnly) {
                [filterCookie addObject:cookie];
            }
        }
    }
    
    // 拼接 JS 代码 对 Client Side 注入Cookie
    NSDictionary *reqheader = [NSHTTPCookie requestHeaderFieldsWithCookies:filterCookie];
    NSString *cookieStr = [reqheader objectForKey:@"Cookie"];
    if (filterCookie.count > 0) {
        for (NSHTTPCookie *cookie in filterCookie) {
            NSTimeInterval expiretime = [cookie.expiresDate timeIntervalSince1970];
            NSString *js = [NSString stringWithFormat:@"document.cookie ='%@=%@;expires=%f';",cookie.name,cookie.value,expiretime];
            WKUserScript *jsscript = [[WKUserScript alloc]initWithSource:js injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
            [self.userContentController addUserScript:jsscript];
        }
    }
    return;
}
```

### 302 跳转无 cookie

可以在跳转回调 `decidePolicyForNavigationAction` 中，手动添加。但是这个方法其实无法区分是否是 302，能做的就是全量的同步。

### cookie 同步问题

WKWebView 也会把 cookie 存储到 `NSHTTPCookiesStorage` 中，但是同步的时间不是立即的。比如在iOS 8上，当页面跳转的时候才会 set。而 iOS 10上，会很快同步（1-2s）。我们可以在请求完成回调中 `decidePolicyForNavigationResponse` 获取 response 的 cookie，并将其设置到 `NSHTTPCookiesStorage` 中。

对于多个 WKWebView 可以共用一个 `WKProcessPool` 可以让 cookie 在多个 webView 之间共享。`WKProcessPool` 只是为了资源共享。没有任何方法和数据可以拿。

## User-Agent

设置 UA 有两种方式，一种是全局的设置 UA，还有一种是设置局部的 UA。

### 全局 UA

设置全局 UA 是通过把包含 `UserAgent` 的字典存入 `NSUserDefaults` 中去：

```objc
 NSDictionary *dictionary = [[NSDictionary alloc] initWithObjectsAndKeys:appUserAgent, @"UserAgent", nil];
[[NSUserDefaults standardUserDefaults] registerDefaults:dictionary];
```

### 自定义 UA

iOS 9 以上提供了自定义 UA 的方式，更加简单，直接成为了 WKWebView 的一个属性：

```
self.webView.customUserAgent = @"WebViewDemo/1.0.0"; 
```

### 获取系统默认 UA

有时候，我们需要自定义 UA 的同时还想要有系统的 UA，即在系统 UA 后添加自己的 UA。这就需要获取到系统的 UA 了。可以通过 `UIWebView` 获取 UA 字符串：

```objc
UIWebView *webView = [[UIWebView alloc] init];
NSString *originalUserAgent = [webView stringByEvaluatingJavaScriptFromString:@"navigator.userAgent"];
NSString *appUserAgent = [NSString stringWithFormat:@"%@-%@", originalUserAgent, customUserAgent];
```

获取到 UA，再通过上面两种方式设置全局或者自定义 UA 即可。



## NSURLProtocol

介绍一下 NSURLProtocol 的使用方式。

### 注册 NSURLProtocol

首先要创建一个 `NSURLProtocol` 的子类：

```objc
@interface MYProtocol : NSURLProtocol
@end
```

然后在任意时候注册：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [NSURLProtocol registerClass:[YXNSURLProtocol class]];
}
```

### 重写 NSURLProtocol 中的几个方法

这里模拟一个真实的业务场景，就是 WKWebView 在首次通过 `loadRequest` 发起 post 的 body 丢失。具体原因见下面的*一些问题*

解决方案就是将请求的 scheme 设置为一个特殊的协议字段，如本例中的 `post`，然后通过 `NSURLProtocol` 拦截。

#### 是否拦截 Request

```objc
+ (BOOL)canInitWithRequest:(NSURLRequest *)request{
    /// 如果 scheme 是 post 那么拦截
    if ([request.URL.scheme isEqualToString:@"post"]) {
        return YES;
    }
    
    /// 如果是已经拦截过的就放行
    if ([NSURLProtocol propertyForKey:@"HasIntercepted" inRequest:request]) {
        return NO;
    }
    return NO;
}
```

未被拦截的 Request 直接放行，拦截的 Request 进入下一个方法

#### 重设 NSURLRequest

被拦截的 `post` 协议来到下面的方法中。这里从 `request.allHTTPHeaderFields`，即 request 的所有头信息中，拿到原本的 scheme 以及原本的 bodyParam。然后生成一个新的 Request，把 body 和 cookie 塞进去，返回这个 Request：

```objc
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    /// 由于 WKWebView 通过 loadRequest 发起的 post 请求 body 会丢失，所以这里通过 NSURLProtocol 拦截，然后自己发出 request
    if ([request.URL.scheme isEqualToString:@"post"]) {
        //获取oldScheme
        NSString *originScheme = request.allHTTPHeaderFields[@"oldScheme"];
        
        NSMutableString *urlString = [NSMutableString stringWithString:request.URL.absoluteString];
        
        NSRange schemeRange = [urlString rangeOfString:request.URL.scheme];
        
        [urlString replaceCharactersInRange:schemeRange withString:originScheme];
        
        //根据新的urlString生成新的request
        NSMutableURLRequest *newRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:urlString]];
        
        //获取bodyParam
        NSString *bodyParam = request.allHTTPHeaderFields[@"bodyParam"];
        NSData *bodyData =[bodyParam dataUsingEncoding:NSUTF8StringEncoding];
        newRequest.HTTPMethod = @"POST";
        newRequest.HTTPBody = bodyData;
        
        //获取cookie
        NSString *cookie = request.allHTTPHeaderFields[@"Cookie"];
        [newRequest addValue:cookie forHTTPHeaderField:@"Cookie"];
        
        [NSURLProtocol setProperty:@YES forKey:@"HasIntercepted" inRequest:newRequest];
        
        return newRequest;
    }
    
    
    return request;
}
```

这里还要注意一点，我们将新生成的 Request 添加一个 `HasIntercepted` 的标记。这样再重新进入 `canInitWithRequest` 的时候就会被直接放行了，防止无限循环。

#### 加载 Request

来到了最后一步，自行创建一个 `NSURLSession` 来实现网络请求：

```objc
- (void)startLoading {
    NSURLSession *session = [NSURLSession sharedSession];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:self.request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if (!error) {
            // 请求成功了，把 response 和 data 都返回回去
            [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageAllowed];
            [self.client URLProtocol:self didLoadData:data];
            [[self client] URLProtocolDidFinishLoading:self];
        }else{
            [self.client URLProtocol:self didFailWithError:error];
        }
    }];
    [task resume];
    self.dataTask = task;
}

- (void)stopLoading {
    [self.dataTask cancel];
}
```

`self.client` 就是操作最后拦截结果的实例，`self.request` 就是上面创建的新的 Request。整个拦截过程就完成了。

### 拦截 WKWebView 的请求

WKWebView 默认是无法被 NSURLProtocol 拦截的，但是我们可以通过私有 Api 实现：

```objc
//注册
[NSURLProtocol registerClass:[MyCustomURLProtocol class]];
//实现拦截功能
Class cls = NSClassFromString(@"WKBrowsingContextController");
SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
if ([(id)cls respondsToSelector:sel]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    [(id)cls performSelector:sel withObject:@"myapp"];
#pragma clang diagnostic pop
}
```

> 上面是为 WKWebView 注册了一个 scheme 为 myapp 的 NSURLProtocol，对于要拦截 http 或者 https 请求，换成相应 scheme 就可以了。

> 针对这个问题，iOS11 推出了新的 API **WKURLSchemeHandler**，能够提供拦截 WKWebView 的功能。



### NSURLProtocol 的应用场景

通过自定义的NSURLProtocol，我们拿到用户请求的request之后，我们可以做很多事情。比如：

1. 自定义请求和响应
2. 网络的缓存处理（H5离线包 和 网络图片缓存）
3. 重定向网络请求
4. 过滤掉一些非法请求

等等...

## 一些问题

### 白屏问题

当WKWebView加载的网页占用内存过大时，会出现白屏现象。解决方案：

```objc
// 当 WKWebView 总体内存占用过大，页面即将白屏的时候，系统会调用上面的回调函数
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView {
    [webView reload];   //刷新就好了
}

```

### WKWebView 上通过 loadRequest 发起的 post 请求 body 数据会丢失

这是因为 WKWebView 有自己单独的一条进程。`loadRequest` 其实是将请求信息从应用进程传递给 WKWebView 所在进程，使其展示的过程。然而，苹果出于进程间通信加快速度的考虑，丢弃了 post 请求的 body 信息。

解决方案是通过 `NSURLProtocol` 拦截 Request，这样 WKWebView 又把 post 请求回传给了 native。发送请求前，把原本 body 中的信息放到 header 中。然后由 `NSURLProtocol` 拦截，生成新的请求，完成数据加载，最后将请求得到的数据返回。

详见上面的 NSURLProtocol 使用介绍，有详细步骤讲解

### WKWebView NSURLProtocol 的 post 请求的 body 为空

上面 loadRequest 是从 app 将 post 请求传给 WKWebView，而此例是由于 NSURLProtocol 拦截，需要把 WKWebView 的 post 请求传递给 app 处理。因此，post 请求的 body 还是会丢失。

所以，解决方式还是同上面 loadRequest 一样。



## 性能优化

优化主要集中在优化 WebView 初始化和减少不必要的请求。

### 全局 WebView 与 WebViewPool

webview 从不存在到存在的过程，系统需要进行一系列初始化。所有后续过程在这段时间完全阻塞。

可以创建一个 WebViewPool 的单例对象，在 load 方法中监听应用启动成功的通知： `UIApplicationDidFinishLaunchingNotification`，初始化 Pool 对象，并且初始化任意个供复用的 WebView 实例，之后的复用很像 TableView Cell 的复用。

之后业务上的所有的 WebView 实例都从 WebViewPool 中拿。WebView 需要增加一个 holder 的弱引用属性指向当前 VC。每次要从 WebViewPool 中取新的 WebView 实例的时候，就要查看一遍哪些 WebView 的 holder 为 nil，表示 WebView 所在的 VC 已经被回收，此时就要把 WebView 状态清空，然后放入复用池中。

当 WebView 需要放回复用池的时候需要做两件事，以达到和浏览器相同的效果

1. 添加一个空白的页面
2. 把该页面之前的浏览记录清空

```objc
// 继承于 WKWebView 的类中
//被回收
- (void)webViewEndReuse{
    // 加载空白页面，空字符串会自动加一个空白页面
    [self loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@""]]];
  
  	// 使用的私有API，所以通过字符串拼接的方式获取方法名
  	// 这个方法会把除了最上面的页面都清空掉。因此，会清空除了刚添加的空白页面的之前所有的页面，使页面恢复到最开始打开时候的模样。
    SEL sel = NSSelectorFromString([NSString stringWithFormat:@"%@%@%@%@", @"_re", @"moveA",@"llIte", @"ms"]);
    if([self.backForwardList respondsToSelector:sel]){
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [self.backForwardList performSelector:sel];
#pragma clang diagnostic pop
    }
  	

}
```

### webView 数据预请求

在客户端初始化WebView的同时，直接由native开始网络请求数据。当页面初始化完成后，向native获取其代理请求的数据。如果此时 native 还没有拿到数据，那么 js 端做一个短暂的轮询。

### 减少不必要的请求

分为前端优化和客户端优化

#### 前端优化

1. **降低请求量**：合并资源，减少 HTTP 请求数，使用 lazyLoad，使用 gzip 压缩，使用 webP 格式
2. **加快请求速度**：预解析 DNS，减少域名数，使用与 Native 一样的域名
3. **缓存**：使用 localStorage，询问是否更新
4. **渲染**：服务端渲染

#### 客户端优化

**NSURLProtocol 拦截资源请求**

对于一些图片资源文件，可以通过 NSURLProtocol 拦截请求，然后查找 native 是否存在缓存，有的话直接返回 NSData，没有的话，通过 native 发起一个请求，缓存并返回 NSData

**基于 LocalWebServer 实现的离线资源加载**

通过 NSURLProtocol 可以实现资源的本地拦截加载。还有一种通过本地起一个 localserver 的方式，直接加载本地的资源。

可以选用如 `CocoaHttpServer` 这样的第三方库，在离线资源所在的目录启动本地服务。这样，网页中的资源，可以直接通过 `http://localhost:[端口号]/someResource.js` 的方式加载。

但是这样会引起 ATS 相关问题，即在 `Safari` 及 `Apple WebKit` 中：在https页面内，不允许http请求存在，否则一概会被block。因此，需要自签名 localhost，使其支持 `https://localhost:[端口号]/someResource.js` ，具体使用可在需要的时候搜索到。

**离线包**

离线包可以预下载，native 根据配置，在某个 节点下载离线包。下载好的离线包后，就可以拦截网络请求，对于离线包已经有的文件，直接读取离线包数据返回，否则走 HTTP 协议缓存逻辑。

```objc
//下载离线包html+css
- (void)requestOfflinePkg {
    NSString *zipName    = @"offline_pkg";
    NSString *zipUrl     = [NSString stringWithFormat:@"http://localhost:9090/source/%@.zip", zipName];
    NSURL    *url        = [NSURL URLWithString:zipUrl];
    NSString *md5        = [self md5:zipUrl];
    NSArray  *pathes     = NSSearchPathForDirectoriesInDomains(NSCachesDirectory,NSUserDomainMask,YES);
    NSString *path       = [pathes objectAtIndex:0];
    NSString *zipPath    = [NSString stringWithFormat:@"%@/zipDownload/%@",path,md5];
    NSString *unzipPath  = [NSString stringWithFormat:@"%@/%@.zip",path,md5];


    NSURLSession *session = [NSURLSession sharedSession];

    NSURLSessionDataTask *task = [session dataTaskWithURL:url completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if(!error) {
            [data writeToFile:unzipPath options:0 error:nil];

            BOOL result = [SSZipArchive unzipFileAtPath:unzipPath toDestination:zipPath];

            //解压缩成功
            if (result) {
                //删除zip
                NSFileManager *fileManager = [NSFileManager defaultManager];
                [fileManager removeItemAtPath:unzipPath error:nil];
            }
        }
    }];

    [task resume];
}
```



## 参考



[移动 H5 首屏秒开优化方案探讨](http://blog.cnbang.net/tech/3477/)

[从零收拾一个hybrid框架（二）-- WebView容器基础功能设计思路](http://awhisper.github.io/2018/03/06/hybrid-webcontainer/)

[[WebView性能、体验分析与优化](https://tech.meituan.com/2017/06/09/webviewperf.html)](https://tech.meituan.com/2017/06/09/webviewperf.html)

[基于 LocalWebServer 实现 WKWebView 离线资源加载](<https://www.jianshu.com/p/a69e77bf680c>)

