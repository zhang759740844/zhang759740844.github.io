title: WKWebView 使用（二）—— 常见问题
date: 2018/8/31 14:07:12  
categories: iOS
tags: 

- 学习笔记

------

记录一下 WKWebView 相关使用

<!--more-->

## Cookie 管理

网上有很多关于 Cookie 丢失时候的方法，使用场景不多，出现问题了看看就好。

### 首次加载无 Cookie

WKWebView 不会在请求的时候自动到 `NSHTTPCookieStorage` 中获取 cookie。

所以需要我们在  `loadRequest` 前，手动在 request header 中设置 Cookie, 解决首个请求 Cookie 带不上的问题:

```objc
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://www.baidu.com"]];
NSArray *cookies = [NSHTTPCookieStorage sharedHTTPCookieStorage].cookies;
//Cookies数组转换为requestHeaderFields
NSDictionary *requestHeaderFields = [NSHTTPCookie requestHeaderFieldsWithCookies:cookies];
//设置请求头
request.allHTTPHeaderFields = requestHeaderFields;
[self.webView loadRequest:request];
```

### 解决后续Ajax请求Cookie丢失问题

主要是给 WKWebView 注入脚本。js 端的 cookie 和 `NSHTTPCookieStorage` 同步。

```objc
/*!
 *  更新webView的cookie
 */
- (void)updateWebViewCookie
{
    // 创建要注入的脚本,每次要加载网页的时候注入
    WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource:[self cookieString] injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
    //添加Cookie
    [self.configuration.userContentController addUserScript:cookieScript];
}

- (NSString *)cookieString
{	
    NSMutableString *script = [NSMutableString string];
    // 先获取到 js 端存在的所有的 cookie 的键
    [script appendString:@"var cookieNames = document.cookie.split('; ').map(function(cookie) { return cookie.split('=')[0] } );\n"];
    for (NSHTTPCookie *cookie in [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookies]) {
        // Skip cookies that will break our script
        if ([cookie.value rangeOfString:@"'"].location != NSNotFound) {
            continue;
        }
        // 比较 native 保存的键 js 端是否包含了，如果不包含，直接把它设置给 js 端。(js 端设置 document.cookie = 'xxx' 并不是直接把之前的都替换掉，而是取并集。可以尝试在 chrome 中操作一下)
        [script appendFormat:@"if (cookieNames.indexOf('%@') == -1) { document.cookie='%@'; };\n", cookie.name, cookie.da_javascriptString];
    }
    return script;
}

@interface NSHTTPCookie (Utils)

- (NSString *)da_javascriptString;

@end

@implementation NSHTTPCookie (Utils)

- (NSString *)da_javascriptString
{
    NSString *string = [NSString stringWithFormat:@"%@=%@;domain=%@;path=%@",
                        self.name,
                        self.value,
                        self.domain,
                        self.path ?: @"/"];
    if (self.secure) {
        string = [string stringByAppendingString:@";secure=true"];
    }
    return string;
}

@end
```

### 跳转新页面时Cookie带不过去问题

打开新页面，对应的就是 safari 增加新的标签页的情景。在 WKWebView 中 `navigationAction.targetFrame` 的 `mainFrame` 属性为NO，表明这个 `WKNavigationAction` 将会新开一个页面。

```objc
//核心方法：
modalPresentationStyle
 修复打开链接Cookie丢失问题

 @param request 请求
 @return 一个fixedRequest
 */
- (NSURLRequest *)fixRequest:(NSURLRequest *)request
{
    NSMutableURLRequest *fixedRequest;
    if ([request isKindOfClass:[NSMutableURLRequest class]]) {
        // 如果这个请求是可变的，那么后面就直接操作这个请求的 cookie
        fixedRequest = (NSMutableURLRequest *)request;
    } else {
        // 如果这个请求是不可变的，那么就深拷贝一份，然后设置 cookie
        fixedRequest = request.mutableCopy;
    }
    //防止Cookie丢失
    NSDictionary *dict = [NSHTTPCookie requestHeaderFieldsWithCookies:[NSHTTPCookieStorage sharedHTTPCookieStorage].cookies];
    if (dict.count) {
        NSMutableDictionary *mDict = request.allHTTPHeaderFields.mutableCopy;
        [mDict setValuesForKeysWithDictionary:dict];
        fixedRequest.allHTTPHeaderFields = mDict;
    }
    return fixedRequest;
}

#pragma mark - WKNavigationDelegate 

- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {

#warning important 这里很重要
    //解决Cookie丢失问题
    NSURLRequest *originalRequest = navigationAction.request;
    [self fixRequest:originalRequest];
    //如果originalRequest就是NSMutableURLRequest, originalRequest中已添加必要的Cookie，可以跳转
    //允许跳转
    decisionHandler(WKNavigationActionPolicyAllow);
    //可能有小伙伴，会说如果originalRequest是NSURLRequest，不可变，那不就添加不了Cookie了，是的，我们不能因为这个问题，不允许跳转，也不能在不允许跳转之后用loadRequest加载fixedRequest，否则会出现死循环，具体的，小伙伴们可以用本地的html测试下。
    
    NSLog(@"%@", NSStringFromSelector(_cmd));
}

#pragma mark - WKUIDelegate

// 跳转到新页面调用
- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures {

#warning important 这里也很重要
    //这里不打开新窗口
    [self.webView loadRequest:[self fixRequest:navigationAction.request]];
    return nil;
}
```

### 本地保存 Cookie

如果还是有 Cookie 丢失的情况，那么可以通过本地保存。

```objc
//比如在 h5 中登录成功的时候，保存Cookie。或者 Native 登录成功后，也可以保存 token，到时候 h5 页面把 token 作为 cookie 传过去。
- (void)saveCookies {
    NSArray *allCookies = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookies];
    for (NSHTTPCookie *cookie in allCookies) {
        if ([cookie.name isEqualToString:DAServerSessionCookieName]) {
            NSDictionary *dict = [[NSUserDefaults standardUserDefaults] dictionaryForKey:DAUserDefaultsCookieStorageKey];
            if (dict) {
                NSHTTPCookie *localCookie = [NSHTTPCookie cookieWithProperties:dict];
                if (![cookie.value isEqual:localCookie.value]) {
                    NSLog(@"本地Cookie有更新");
                }
            }
            [[NSUserDefaults standardUserDefaults] setObject:cookie.properties forKey:@"someKey"];
            [[NSUserDefaults standardUserDefaults] synchronize];
            break;
        }
    }
}
```

在读取时，如果没有则添加：

```objc
@implementation NSHTTPCookieStorage (Utils)

+ (void)load
{
    // hook 获取 cookies 的方法
    class_methodSwizzling(self, @selector(cookies), @selector(da_cookies));
}

- (NSArray<NSHTTPCookie *> *)da_cookies
{
    NSArray *cookies = [self da_cookies];
    BOOL isExist = NO;
    for (NSHTTPCookie *cookie in cookies) {
        if ([cookie.name isEqualToString:DAServerSessionCookieName]) {
            isExist = YES;
            break;
        }
    }
    if (!isExist) {
        //CookieStroage中添加
        NSDictionary *dict = [[NSUserDefaults standardUserDefaults] dictionaryForKey:@"someKey"];
        if (dict) {
            NSHTTPCookie *cookie = [NSHTTPCookie cookieWithProperties:dict];
            [[NSHTTPCookieStorage sharedHTTPCookieStorage] setCookie:cookie];
            NSMutableArray *mCookies = cookies.mutableCopy;
            [mCookies addObject:cookie];
            cookies = mCookies.copy;
        }
    }
    return cookies;
}

@end
```

## User-Agent

### 全局的 UA

```objc
//最好在AppDelegate中就提前设置
@implementation AppDelegate


- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    
    //设置自定义UserAgent
    [self setCustomUserAgent];
    return YES;
}

- (void)setCustomUserAgent
{
    //get the original user-agent of webview
    UIWebView *webView = [[UIWebView alloc] initWithFrame:CGRectZero];
    NSString *oldAgent = [webView stringByEvaluatingJavaScriptFromString:@"navigator.userAgent"];
    //add my info to the new agent
    NSString *newAgent = [oldAgent stringByAppendingFormat:@" %@", @"WebViewDemo/1.0.0"];
    //regist the new agent
    NSDictionary *dictionnary = [[NSDictionary alloc] initWithObjectsAndKeys:newAgent, @"UserAgent", newAgent, @"User-Agent", nil];
    // 全局设置 UA
    [[NSUserDefaults standardUserDefaults] registerDefaults:dictionnary];
}

@end
```

虽然这里是通过 `UIWebView` 设置的，但是通过 `NSUserDefaults` 设置自定义`UserAgent`，可以同时作用于 `UIWebView` 和 `WKWebView` 。

### 局部的 UA

关于自定义UA，`WKWebView`提供Api，前文中也说明过，就是调用`customUserAgent`属性：

```objc
self.webView.customUserAgent = @"WebViewDemo/1.0.0"; 
```

`WKWebView`的`customUserAgent`属性，优先级高于`NSUserDefaults`，当同时设置时，显示`customUserAgent`的值

如果不想覆盖原有的值, 可以这样:

```objc
webView.evaluateJavaScript("navigator.userAgent") { [weak webView] (info, error) in
    // 获取默认值
    if var userAgent = info as? String {
        // 添加自定义的内容
        if userAgent.hasSuffix("/ios-app-cgyc") == false {
            userAgent += "/ios-app"
        }

        if #available(iOS 9.0, *) {
            webView?.customUserAgent = userAgent
        } else {
            // Fallback on earlier versions
        }

        // 重要：加载网页
        let request = URLRequest(url: ul)
        webView?.load(request)
    }
}
```

注意，一定要在设置完 UA 之后，再加载网页

## NSURLProtocol

`NSURLProtocol` 主要可以做一些缓存操作。使用 NSURLProtocol 对网络请求进行过滤，如果是特定的 html、css、js 等请求和图片请求，就在本地缓存数据中进行查找，命中缓存则使用本地数据包装成网络请求的响应数据进行回传，否则才执行对应的网络请求。这样即使在网络可用的情况下，也会优先使用本地缓存数据，不仅可以完成页面的离线展示，还有效地加快了网络可用时页面的加载速度。

### h5加载本地相册中的图片

有这样的一个需求，就是 WKWebView 中要加载本地相册中的一张图片，然后展示。h5 是无法直接加载本地的资源的，所以需要通过 NSURLProtocol 拦截 h5 的请求。

本地相册选择后，我们可以拿到图片的 UIImage 对象。我们需要先把她保存到本地的地址下：

```objc
UIImage *image = [info  objectForKey:UIImagePickerControllerOriginalImage];
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask, YES);
    NSString *filePath = [[paths objectAtIndex:0] stringByAppendingPathComponent:[NSString stringWithFormat:@"%@.png",timeString]];  //保存到本地
    [UIImagePNGRepresentation(image) writeToFile:filePath atomically:YES];

```

然后，把保存的 `filePath`  传递给 webView 调用 h5 约定好的方法：

```objc
NSString *str = [NSString stringWithFormat:@"%@",filePath];
[picker dismissViewControllerAnimated:YES completion:^{
        // oc 调用js 并且传递图片路径参数
        [self.webview evaluateJavaScript:[NSString stringWithFormat:@"getImg('%@')",str] completionHandler:^(id _Nullable data, NSError * _Nullable error) {
        }];
    }];
}

```

h5 在方法中要发出一个网络请求。然后被 Native 拦截。我们需要创建一个 `NSURLProtocol` 拦截器:

```objc
@interface MyCustomURLProtocol : NSURLProtocol
@end
    
@implementation MyCustomURLProtocol
+ (BOOL)canInitWithRequest:(NSURLRequest*)theRequest{
    if ([theRequest.URL.scheme caseInsensitiveCompare:@"myapp"] == NSOrderedSame) {
        return YES;
    }
    return NO;
}

+ (NSURLRequest*)canonicalRequestForRequest:(NSURLRequest*)theRequest{
    return theRequest;
}

- (void)startLoading{
    NSURLResponse *response = [[NSURLResponse alloc] initWithURL:[self.request URL] MIMEType:@"image/png" expectedContentLength:-1 textEncodingName:nil];
    NSString *imagePath = [self.request.URL.absoluteString componentsSeparatedByString:@"myapp://"].lastObject;
    NSData *data = [NSData dataWithContentsOfFile:imagePath];
    [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    [[self client] URLProtocol:self didLoadData:data];
    [[self client] URLProtocolDidFinishLoading:self];
}
- (void)stopLoading{
}
@end

```

我们需要在 ViewController 初始化的时候，注册这个拦截器。WKWebView 的 `NSURLProtocol` 使用起来写法比较特殊：

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

> 上面是为 WKWebView 注册了一个 scheme 为 myapp 的 `NSURLProtocol`，对于要拦截 http 或者 https 请求，换成相应 scheme 就可以了。



## 一些问题

### 白屏问题

当WKWebView加载的网页占用内存过大时，会出现白屏现象。解决方案：

```objc
// 当 WKWebView 总体内存占用过大，页面即将白屏的时候，系统会调用上面的回调函数
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView {
    [webView reload];   //刷新就好了
}

```

### WKWebView NSURLProtocol 的 post 请求的 body 为空

由于 WKWebView 在独立进程里执行网络请求。一旦注册 http(s) scheme 后，网络请求将从 Network Process 发送到 App Process，这样 NSURLProtocol 才能拦截网络请求。在 webkit2 的设计里使用 MessageQueue 进行进程之间的通信，Network Process 会将请求 encode 成一个 Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode 的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了。

因此，**如果通过 registerSchemeForCustomProtocol 注册了 http(s) scheme, 那么由 WKWebView 发起的所有 http(s)请求都会通过 IPC 传给主进程 NSURLProtocol 处理，导致 post 请求 body 被清空**；

### WKWebView 上通过 loadRequest 发起的 post 请求 body 数据会丢失

一般加载一个网页请求为啥要用 post 咧。所以简单点用 get 就好啦。

但是如果还是想要通过 post 发起请求的话。假如想通过-[WKWebView loadRequest:]加载 post 请求 request1。需要进行以下步骤：

1. 不直接把 post 请求的信息放到 body 中，而是创建新的请求 request2。同时将 request1 的 body 字段复制到 request2 的 header 中（WebKit 不会丢弃 header 字段）。
2. 然后通过`-[WKWebView loadRequest:]` 加载新的 post 请求 request2。
3. 通过 `+[WKBrowsingContextController registerSchemeForCustomProtocol:]` 注册 scheme
4. 注册 NSURLProtocol 拦截请求,替换请求 scheme, 由 native 端生成新的请求 request3。
5. 将 request2 header的body 字段复制到 native 发起的请求 request3 的 body 中，并使用 NSURLConnection 加载 request3，最后通过 NSURLProtocolClient 将加载结果返回 WKWebView;