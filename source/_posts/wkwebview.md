title: WKWebView 使用（一）—— 基本用法
date: 2018/7/31 14:07:12  
categories: iOS
tags: 

 - 学习笔记
	
---

WKWebView 的简单用法

<!--more-->

## 基本用法

### 创建

```objective-c
- (instancetype)initWithFrame:(CGRect)frame configuration:(WKWebViewConfiguration *)configuration;
```

比`UIWebView`多了个`configuration`，这个配置可以设置很多东西。具体查看`WKWebViewConfiguration.h`，可以配置js是否支持，画中画是否开启等，这里主要讲两个比较常用的属性:

#### websiteDataStore

`WKWebView` 拥有自己的私有存储，它的一些缓存等数据都存在`websiteDataStore`中。

```objc
@property (nonatomic, strong) WKWebsiteDataStore *websiteDataStore;
```

具体增删改查就可以通过`WKWebsiteDataStore.h`中提供的方法，这里不多说，一般用的时候比较少，真的要清除缓存，简单粗暴的方法是删除沙盒目录中的Cache文件夹。

#### userContentController

这个属性很重要，js与oc交互，以及注入js都会用到。

```objc
@property (nonatomic, strong) WKUserContentController *userContentController;
```

查看`WKUserContentController`的头文件，你会发现它有如下几个方法：

```objc
@interface WKUserContentController : NSObject <NSCoding>
//读取添加过的脚本
@property (nonatomic, readonly, copy) NSArray<WKUserScript *> *userScripts;
//添加脚本
- (void)addUserScript:(WKUserScript *)userScript;
//删除所有添加的脚本
- (void)removeAllUserScripts;
//通过window.webkit.messageHandlers.<name>.postMessage(<messageBody>) 来实现js->oc传递消息，并添加handler
- (void)addScriptMessageHandler:(id <WKScriptMessageHandler>)scriptMessageHandler name:(NSString *)name;
//删除handler
- (void)removeScriptMessageHandlerForName:(NSString *)name;
@end
```

#### 基本创建

```objc
WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
WKUserContentController *controller = [[WKUserContentController alloc] init];
configuration.userContentController = controller;
self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:configuration];
self.webView.allowsBackForwardNavigationGestures = YES; //允许右滑返回上个链接，左滑前进
self.webView.allowsLinkPreview = YES; //允许链接3D Touch
self.webView.customUserAgent = @"WebViewDemo/1.0.0"; //自定义UA，UIWebView就没有此功能，后面会讲到通过其他方式实现
self.webView.UIDelegate = self;
self.webView.navigationDelegate = self;
[self.view addSubview:self.webView];
```

### 动态注入js

通过给`userContentController`添加`WKUserScript`，可以实现动态注入js。比如我先注入一个脚本，给每个页面添加一个Cookie

```objc
//注入一个Cookie
WKUserScript *newCookieScript = [[WKUserScript alloc] initWithSource:@"document.cookie = 'DarkAngelCookie=DarkAngel;'" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
[controller addUserScript:newCookieScript];
```

然后再注入一个脚本，每当页面加载，就会alert当前页面cookie:

```objc
//创建脚本
WKUserScript *cookieScript = [[WKUserScript alloc] initWithSource:@"alert(document.cookie);" injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:NO];
//添加脚本
[controller addUserScript:script];
```

注入的js source可以是任何js字符串，也可以js文件。比如你本地可能就会有一个`native_functions.js`，你可以通过以下的方式添加：

```objc
NSString *jsSource = [NSString stringWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"native_functions" ofType:@"js"] encoding:NSUTF8StringEncoding error:nil];
//添加自定义的脚本
WKUserScript *js = [[WKUserScript alloc] initWithSource:jsSource injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:NO];
[self.configuration.userContentController addUserScript:js];
```

### 加载

```objc
- (nullable WKNavigation *)loadRequest:(NSURLRequest *)request;
- (nullable WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL 
- (nullable WKNavigation *)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;
```

加载主 bundle 中的一个html需要使用`loadRequest:`方法。`loadReqest` 这种方式会把当前load的这个html文件的路径作为baseURL，以此寻找其他资源。

```objc
[self.webView loadRequest:[NSURLRequest requestWithURL:[NSURL fileURLWithPath:[[NSBundle mainBundle] pathForResource:@"test" ofType:@"html"]]]];
```

但是要注意，如果**加载本地沙盒中的文件**，这个方法只能加载本地 `tmp` 目录下的文件，需要先将本地 HTML 文件的数据 copy 到 `tmp` 目录中，然后再使用 `loadRequest` 来加载。iOS9以上 `[WKWebView loadFileURL:allowingReadAccessToURL:]` 加载任意位置的文件。

`loadHTMLString:baseURL:` 直接加载 html 字符串。`baseURL` 是 html 中资源的路径：

```objc
NSString *html = @"....";
// 加载 mainBundle 下的资源 `[[NSBundle mainBundle] resourcePath]` 获取 mainBundle 的路径字符串
[self.wkWebView loadHTMLString:html baseURL:[NSURL fileURLWithPath:[[NSBundle mainBundle] resourcePath]]];
```

### 代理

`UIWebView`的代理协议拆成了一个跳转的协议和一个关于UI的协议：

```objc
@protocol WKNavigationDelegate; //类似于UIWebView的加载成功、失败、是否允许跳转等
@protocol WKUIDelegate; //主要是一些alert、打开新窗口之类的
```

#### WKNavigationDelegate

常用方法：

```objc
//下面这2个方法共同对应了UIWebView的 - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
//先：针对一次action来决定是否允许跳转，action中可以获取request，允许与否都需要调用decisionHandler，比如decisionHandler(WKNavigationActionPolicyCancel);
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler；
//后：根据response来决定，是否允许跳转，允许与否都需要调用decisionHandler，如decisionHandler(WKNavigationResponsePolicyAllow);
- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler;

//开始加载，对应UIWebView的- (void)webViewDidStartLoad:(UIWebView *)webView;
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(null_unspecified WKNavigation *)navigation;

//加载成功，对应UIWebView的- (void)webViewDidFinishLoad:(UIWebView *)webView;
- (void)webView:(WKWebView *)webView didFinishNavigation:(null_unspecified WKNavigation *)navigation;

//加载失败，对应UIWebView的- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
- (void)webView:(WKWebView *)webView didFailNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error;
```

##### 示例：控制哪些站点可以被访问

```swift
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
    if let host = navigationAction.request.url?.host {
        if host == "www.apple.com" {
            decisionHandler(.allow)
            return
        }
    }
    decisionHandler(.cancel)
}
```



#### WKUIDelegate

当 JS 中调用 alert 等方法的时候，会来到这几个代理方法中，进行自定义操作。可以使用这个方式来进行 JS 向 Native 传递参数执行方法。

```objc
/*  警告 */
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:message ? message : @"" preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:^(UIAlertAction *action) {
        completionHandler();
    }]];
    [vc presentViewController:alert animated:YES completion:NULL];
}
///** 确认框 */
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL result))completionHandler{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:message ? message : @"" preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(YES);
    }]];
    [alert addAction:[UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(NO);
    }]];
    [vc presentViewController:alert animated:YES completion:NULL];
}
/**  输入框 */
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * __nullable result))completionHandler{
    [alert addTextFieldWithConfigurationHandler:^(UITextField * _Nonnull textField) {
        textField.textColor = [UIColor blackColor];
        textField.placeholder = defaultText ? defaultText : @"";
    }];
    [alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler([[alert.textFields lastObject] text]);
    }]];
    [alert addAction:[UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(nil);
    }]];
    [vc presentViewController:alert animated:YES completion:NULL];
}
```

### 新属性

```objc
@property (nullable, nonatomic, readonly, copy) NSString *title;    //页面的title，终于可以直接获取了
@property (nullable, nonatomic, readonly, copy) NSURL *URL;     //当前webView的URL
@property (nonatomic, readonly, getter=isLoading) BOOL loading; //是否正在加载
@property (nonatomic, readonly) double estimatedProgress;   //加载的进度
@property (nonatomic, readonly) BOOL canGoBack; //是否可以后退，跟UIWebView相同
@property (nonatomic, readonly) BOOL canGoForward;  //是否可以前进，跟UIWebView相同
```

这些属性都很有用，而且**支持KVO**，所以我们可以通过KVO观察这些值的变化，以便于我们做出最友好的交互。

## OC JS 交互

### OC -> JS

```objc
//执行一段js，并将结果返回，如果出错，error则不为空
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id result, NSError * _Nullable error))completionHandler;
```

示例，比如我想获取页面中的`title`，除了直接`self.webView.title`外，还可以通过这个方法：

```objc
[self.webView evaluateJavaScript:@"document.title" completionHandler:^(id _Nullable title, NSError * _Nullable error) {
        NSLog(@"调用evaluateJavaScript异步获取title：%@", title);
}];
```

### JS -> OC

#### URL 拦截

通过自定义Scheme，在链接激活时，拦截该URL，拿到参数，调用OC方法：

```objc
// 在HTML中写上A标签直接填写假请求地址
<a href="myScheme://login?username=12323123&code=892845">短信验证登录</a>
// 在JS中用location.href跳转
location.href = 'wakaka://wahahalalala/action?param=paramobj'
// 在JS中创建一个iframe，然后插入dom之中进行跳转
$('body').append('<iframe src="' + 'wakaka://wahahalalala/action?param=paramobj' + '" style="display:none"></iframe>');

- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    //可以通过navigationAction.navigationType获取跳转类型，如新链接、后退等
    NSURL *URL = navigationAction.request.URL;
    //判断URL是否符合自定义的URL Scheme
    if ([URL.scheme isEqualToString:@"myScheme"]) {
        //根据不同的业务，来执行对应的操作，且获取参数
        if ([URL.host isEqualToString:@"login"]) {
            NSString *param = URL.query;
            NSLog(@"短信验证码登录, 参数为%@", param);
            decisionHandler(WKNavigationActionPolicyCancel);
            return;
        }
    }
    decisionHandler(WKNavigationActionPolicyAllow);
    NSLog(@"%@", NSStringFromSelector(_cmd));
}
```

WKWebView 拦截：
```objc
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    //1 根据url，判断是否是所需要的拦截的调用 判断协议/域名
    if (是){
      //2 取出路径，确认要发起的native调用的指令是什么
      //3 取出参数，拿到JS传过来的数据
      //4 根据指令调用对应的native方法，传递数据
      //确认拦截，拒绝WebView继续发起请求
        decisionHandler(WKNavigationActionPolicyCancel);
    }else{
        decisionHandler(WKNavigationActionPolicyAllow);
    }
    return YES;
}
```

假跳转的缺点：
- 丢失消息。在同一个运行逻辑内快速的连续发送两个通信请求，那么后面的消息将收不到。
- URL长度限制。

#### scriptMessageHandler

在OC中添加一个scriptMessageHandler，则会在`all frames`中添加一个js的function： `window.webkit.messageHandlers.<name>.postMessage(<messageBody>)` 。那么当我在OC中通过如下的方法添加了一个handler，如

```objc
//配置对象注入
[self.webView.configuration.userContentController addScriptMessageHandler:self name:@"nativeObject"];

```

> 这里不能直接传入 self，会被 `WKUserContentController` 强引用，而不能销毁。所以需要创建一个代理类对象，类似于处理 NSTimer。

当我在js中调用下面的方法时:

```objc
//准备要传给native的数据，包括指令，数据，回调等
var data = {
    action:'xxxx',
    params:'xxxx',
    callback:'xxxx',
};
//传递给客户端
window.webkit.messageHandlers.nativeObject.postMessage(data);
```

> 注意，`postMessage` 方法要求必须要有一个参数，即使是一个空对象，也要写成 `postMessage({})`，否则 native 无法收到消息。

在OC中将会收到`WKScriptMessageHandler`的回调

```objc
-(void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
    //1 解读JS传过来的JSValue  data数据
    NSDictionary *msgBody = message.body;
    //2 取出指令参数，确认要发起的native调用的指令是什么
    //3 取出数据参数，拿到JS传过来的数据
    //4 根据指令调用对应的native方法，传递数据
}
```

>  我们可以把js的function转换为字符串，再传递给OC。再让 OC 通过 `evaluateJavaScript:completionHandler:` 调用，将结果传出。就实现了 js->oc->js->oc 的回调

记得在适当的地方调用 `removeScriptMessageHandler`:

```objc
- (void)dealloc {
    //移除对象注入
    [self.webView.configuration.userContentController removeScriptMessageHandlerForName:@"nativeObject"]; 
}
```

#### WKUIDelegate

前面说到，`WKUIDelegate` 协议中的几个弹窗方法，可以用来实现 JS2OC 的通信。前面几种 JS2OC 的通信的 callback 都需要 OC自己调用 OC2JS 的异步方法返回。而使用 `WKUIDelegate` 可以同步返回结果。

**JS 端调用**

```js
var data = {
    action:'xxxx',
    params:'xxxx',
    callback:'xxxx',
};
var jsonData = JSON.stringify([data]);
//发起弹框
prompt(jsonData);
```

 **客户端拦截：**

```objc
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable result))completionHandler{
    //1 根据传来的字符串反解出数据，判断是否是所需要的拦截而非常规H5弹框
    if (是){
        //2 取出指令参数，确认要发起的native调用的指令是什么
        //3 取出数据参数，拿到JS传过来的数据
        //4 根据指令调用对应的native方法，传递数据
        //直接返回JS空字符串
        completionHandler(@"");
    }else{
        //直接返回JS空字符串
        completionHandler(@"");
    }
}
```

这里 `prompt` 和 `defaultText` 就是 JS 端传来的数据。通过 `completionHandler()` 就可以同步将值返回了。js 端调用接收：

```js
let result = window.prompt(somePrompt, someDefaultText)
```

## JavaScriptCore

JavaScriptCore是苹果Safari浏览器的JavaScript引擎

### OC 调用 JS

OC 调用 JS 可以直接使用 `evaluteScript` 方法：

```objc 
#import <JavaScriptCore/JavaScriptCore.h>
int main() {
    JSContext *context = [[JSContext alloc] init];
    JSValue *value = [context evaluateScript:@"2 + 2"];
    NSLog(@"2 + 2 = %d", [value toInt32]);
    return 0
}
```

#### JSContext

- JSContext 是JS代码的执行环境。JSContext 为JS代码的执行提供了上下文环境，通过jSCore执行的JS代码都得通过JSContext来执行。
- JSContext对应于一个 JS 中的全局对象。JSContext对应着一个全局对象，相当于浏览器中的window对象，JSContext中有一个GlobalObject属性，实际上JS代码都是在这个GlobalObject上执行的，但是为了容易理解，可以把JSContext等价于全局对象。我们甚至可以直接在 JSContext 上拿 GlobalObject 的属性。

```objc
JSValue *value = [context evaluateScript:@"var a = 1+2*3;"];

NSLog(@"a = %@", [context objectForKeyedSubscript:@"a"]);
NSLog(@"a = %@", [context.globalObject objectForKeyedSubscript:@"a"]);
NSLog(@"a = %@", context[@"a"]);

/
Output:

a = 7
a = 7
a = 7

```



#### JSValue

- JSValue 是对 JS 值的包装。JS中的值拿到OC中是不能直接用的，需要包装一下
- JSValue存在于JSContext中。JSValue是不能独立存在的，它必须存在于某一个JSContext中。
- JSValue对其对应的JS值和其所属的JSContext对象都是强引用的关系。因为jSValue需要这两个东西来执行JS代码，所以JSValue会一直持有着它们。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jscore1.png?raw=true)

示例：

js 代码：

```js
// 计算阶乘
var factorial = function (n) {
    if (n < 0) return
    if (n === 0) return 1
    return n * factortial(n-1)
}
```

oc 调用：

```objc
NSString *factorialScript = [self loadJSFromBundle];
JSContext *context = [[JSContext alloc] init];
[context evaluteScript: factorialScript];
JSValue *function = context[@"factorial"];
JSValue *result = [function callWithArguments:@[@5]];
NSLog(@"factorial(5) = %d", [result toInt32]); 
```

从 Bundle 中拿到 js 代码执行，从 context 中拿到方法，然后通过 `callWithArguments` 执行，执行后得到 JSValue 的值。

### JS 调用 OC

两种方式可以让 OC 暴露方法给 JS：

- Block：可以将 OC 中的单个方法暴露给 JS 调用。
- JSExport协议：可以将OC的中某个对象直接暴露给JS使用

#### Block

JSCore会自动将这个Block包装成一个JS方法：

```objc
context[@"makeNSColor"] = ^(NSDictorary * colors){
    float r = [colors[@"red"] floatValue];
    float g = [colors[@"green"] floatValue];
    float b = [colors[@"blue"] floatValue];
    return [NSColor colorWithRed:(r / 255.0) green:(g / 255.0) blue:(b / 255) alpha:1.0];
}
```

js 端就可以直接调用这个 `makeNSColor` 方法了。运行得到一个 NSColor 对象，在 js 中表现为一个 object。

如果我们在 js 端再对这个方法封装一下：

```js
var colorForWord = function (word) {
    return makeNSColor(word)
}
```

在 oc 端调用的示意图如下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jscore2.png?raw=true)

`OC Caller`去调用这个`colorForWrod`函数，因为`colorForWrod`函数接收的是一个`String`类型那个参数word，`OC Caller`传过去的是一个`NSString`类型的参数，JSCore转换成对应的`String`类型。然后`colorForWrod`函数继续向下调用，就像上面说的，知道其拿到返回的`wrapper Object`，它将`wrapper Object`返回给调用它的`OC Caller`，JSCore又会在这时候把`wrapper Object`转成JSValue类型，最后再OC中通过对JSValue调用对应的转换方法，即可拿到里面包装的值，这里我们调用`- toObject`方法，最后会得到一个`NSColor`对象，即从最开始那个暴露给JS的Block中返回的对象。

##### 注意点

1. 不要在Block中直接使用JSValue。
2. 不要在Block中直接使用JSContext。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jscore3.png?raw=true)

Block会强引用它里面用到的外部变量，如果直接在Block中使用JSValue的话，那么这个JSvalue就会被这个Block强引用，而每个JSValue都是强引用着它所属的那个JSContext的，这是前面说过的，而这个Block又是注入到这个Context中，所以这个Block会被context强引用，这样会造成循环引用，导致内存泄露。不能直接使用JSContext的原因同理。所以建议把JSValue当做参数传到Block中，而不是直接在Block内部使用，这样Block就不会强引用JSValue了。

针对第二点，可以使用`[JSContext currentContext] `方法来获取当前的Context。

#### JSExport 协议

通过JSExport 协议可以很方便的将OC中的对象暴露给JS使用，且在JS中用起来就和JS对象一样。

声明一个自定义的协议并继承自JSExport协议。然后当你把实现这个自定义协议的对象暴露给JS时，JS就能像使用原生对象一样使用OC对象了：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jscore4.png?raw=true)

## WebViewJavaScriptBridge 源码解析

### 基本用法

1. 导入头文件，声明一个 `WebViewJavascriptBridge` 属性：

```objc
#import "WebViewJavascriptBridge.h"
...
@property WebViewJavascriptBridge* bridge;
```

2. 将 wkwebview 设置给 bridge

```objc
self.bridge = [WebViewJavascriptBridge bridgeForWebView:webView];
```

3. 在 Objective-C 中注册 handler 和调用 JavaScript 中的 handler：

```objc
[self.bridge registerHandler:@"ObjC Echo" handler:^(id data, WVJBResponseCallback responseCallback) {
    NSLog(@"ObjC Echo called with: %@", data);
    responseCallback(data);
}];
```

4. 设置 bridge 的代理为任意你操作的控制器。这样，所有 webView 相关的方法在经过 bridge 的预处理后，都将操作权转移给了使用者：

```objc
[self.bridge setWebViewDelegate:self];
```

5. 复制下面的 `setupWebViewJavascriptBridge` 函数到你的 JavaScript 代码中。该方法用来初始化 js 端的 bridge：

```objc
function setupWebViewJavascriptBridge(callback) {
    if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
    if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
    window.WVJBCallbacks = [callback];
    var WVJBIframe = document.createElement('iframe');
    WVJBIframe.style.display = 'none';
    WVJBIframe.src = 'https://__bridge_loaded__';
    document.documentElement.appendChild(WVJBIframe);
    setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```

6. 调用 `loadRequest`，在执行 JS 代码的时候会调用 `setupWebViewJavascriptBridge` 函数。使用 `bridge` 来注册 handler 和调用 Objective-C 中的 handler：

```objc
setupWebViewJavascriptBridge(function(bridge) {
    bridge.registerHandler('JS Echo', function(data, responseCallback) {
        console.log("JS Echo called with:", data)
        responseCallback(data)
    })
    bridge.callHandler('ObjC Echo', {'key':'value'}, function responseCallback(responseData) {
        console.log("JS received response:", responseData)
    })
})
```

至此，native 端和 js 端都有了一个 bridge，可以相互调用。

7. JS 端主动调用 Native:

```js
bridge.callHandler('ObjC Echo', {'key':'value'}, function responseCallback(responseData) {
    console.log("JS received response:", responseData)
})
```

8. Native 主动调用 JS：

```objc
[self.bridge callHandler:@"JS Echo" data:nil responseCallback:^(id responseData) {
    NSLog(@"ObjC received response: %@", responseData);
}];
```

### 结构

代码分为三个部分：

- 外层调用接口：WebViewJavascriptBridge && WKWebViewJavascriptBridge
- 具体实现： WebViewJavascriptBridgeBase
- JS 层： WebViewJavascriptBridge_JS

可以看得出来，对于一个给别人使用的类库的实现。给外部使用的，只要暴露出寥寥几个方法即可。而具体的实现应该放到另外的具体实现的文件中去，以此区分职责。

### 源码解析

#### 初始化

```objc
// WKWebViewJavaScriptBridge.m
+ (instancetype)bridgeForWebView:(WKWebView*)webView {
    WKWebViewJavascriptBridge* bridge = [[self alloc] init];
    [bridge _setupInstance:webView];
    [bridge._base reset];
    return bridge;
}

- (void) _setupInstance:(WKWebView*)webView {
    _webView = webView;
    _webView.navigationDelegate = self;
    _base = [[WebViewJavascriptBridgeBase alloc] init];
    _base.delegate = self;
}

// WebViewJavascriptBridgeBase.m
- (id)init {
    if (self = [super init]) {
        self.messageHandlers = [NSMutableDictionary dictionary];
        self.startupMessageQueue = [NSMutableArray array];
        self.responseCallbacks = [NSMutableDictionary dictionary];
        _uniqueId = 0;
    }
    return self;
}

- (void)reset {
    self.startupMessageQueue = [NSMutableArray array];
    self.responseCallbacks = [NSMutableDictionary dictionary];
    _uniqueId = 0;
}
```

初始化做的事情不是很多，主要就是生成了一个具体的处理逻辑的实例 `WebViewJavascriptBridgeBase` ，然后 `WKWebViewJavascriptScriptBridge` 将 WKWebView 的代理设置为了自己，虽然后面还是会转发给 `_base` 的相应处理方法处理。

`messageHandlers`,`startupMessageQueue` 和 `responseCallbacks` 后面会讲到。

#### 注册与移除提供给 JS 的 OC 方法

```objc
typedef void (^WVJBResponseCallback)(id responseData);
typedef void (^WVJBHandler)(id data, WVJBResponseCallback responseCallback);

- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}

- (void)removeHandler:(NSString *)handlerName {
    [_base.messageHandlers removeObjectForKey:handlerName];
}
```

代码非常简单，其实就是把提供给 JS 调用的方法的方法名和处理 block 保存在了 `_base` 中的 `messageHandlers` 字典中。

处理 block 的第一个参数为 JS 传递给 OC 的参数，第二个参数为 OC 处理完 JS 的调用后对 JS 的回调。

#### JS 注册

注册好提供给 JS 的方法后，OC 会执行 `loadRequest` 真正的执行 js 代码。我们在 js 中注入的初始化方法 `setupWebViewJavascriptBridge` 就会被执行:

```js
function setupWebViewJavascriptBridge (callback) {
  // 第一次调用的时候 window.WebViewJavascriptBridge 和 WVJBCallbacks 还没有初始化好，所以不会调用
  if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge) }
  if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback) }
  window.WVJBCallbacks = [callback]
    
  // 开启一个 iframe，加载这段 URL 'wvjbscheme://__BRIDGE_LOADED__'
  // 其目的是为了触发 WebViewJavascriptBridge_JS.m 文件内容的加载
  var WVJBIframe = document.createElement('iframe')
  WVJBIframe.style.display = 'none'
  WVJBIframe.src = 'https://__bridge_loaded__'
  document.documentElement.appendChild(WVJBIframe)
  setTimeout(function () { document.documentElement.removeChild(WVJBIframe) }, 0)
}

setupWebViewJavascriptBridge(function (bridge) {
  var uniqueId = 1
  bridge.registerHandler('testJavascriptHandler', function (data, responseCallback) {
    var responseData = { 'Javascript Says': 'Right back atcha!' }
    responseCallback(responseData)
  })
})
```

这里主要做了两件事：

1. 将传进来的 js 初始化成功后需要执行的方法，保存到 `window.WVJBCallbacks` 中，等到后面 JS 端的 bridge 初始化成功后，再取出来调用
2. 通过添加一个 iframe 加载初始化链接 `https://__bridge_loaded__`，调起原生，然后再移除这个 iframe

#### native 拦截 iframe 的 request

js 端为了初始化 bridge，通过 iframe 发起了一个 `https://__bridge_loaded__` 的请求。native 端会受到相应的跳转回调：

```objc
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    if (webView != _webView) { return; }
    NSURL *url = navigationAction.request.URL;
    __strong typeof(_webViewDelegate) strongDelegate = _webViewDelegate;

    if ([_base isWebViewJavascriptBridgeURL:url]) {
        if ([_base isBridgeLoadedURL:url]) {
            // 初始化 bridge
            [_base injectJavascriptFile];
        } else if ([_base isQueueMessageURL:url]) {
            [self WKFlushMessageQueue];
        } else {
            [_base logUnkownMessage:url];
        }
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:decidePolicyForNavigationAction:decisionHandler:)]) {
        [_webViewDelegate webView:webView decidePolicyForNavigationAction:navigationAction decisionHandler:decisionHandler];
    } else {
        decisionHandler(WKNavigationActionPolicyAllow);
    }
}
```

在这个方法中，通过 `[_base isBridgeLoadedURL: url]` 判断是否是制定的加载 bridge 专用的 URL，来决定是否注入 JS 代码。`https://__bridge_loaded__` 这个 URL 就是用来注入 JS 的，会执行 `[_base injectJavascriptFile]` 方法。

#### 注入 JS 文件，初始化 JS 端 bridge

##### native 调用 JS 初始化

```objc
- (void)injectJavascriptFile {
    NSString *js = WebViewJavascriptBridge_js();
    // 执行初始化 JS 端 bridge 的 js 代码
    [self _evaluateJavascript:js];
    // 如果当前已有消息队列则遍历并分发消息，之后清空消息队列
    if (self.startupMessageQueue) {
        NSArray* queue = self.startupMessageQueue;
        self.startupMessageQueue = nil;
        for (id queuedMessage in queue) {
            [self _dispatchMessage:queuedMessage];
        }
    }
}
```

这个方法中，主要就是执行 `WebViewJavascriptBridge_js` 中准备好的 JS。

##### 初始化 JSBridge 中的各个变量

WebViewJavascriptBridge_js` 中保存了几个变量：

```js
// 用来像 native 发送请求的 iframe 实例
var messagingIframe;
// 要发送的各个消息的数组，每个对象的结构为 {handlerName: 'xxx', data: {}, responseId: 1}
var sendMessageQueue = [];
// 保存 handlerName 以及对应的实现方法
var messageHandlers = {};

// native 和 js 端统一的事件 scheme 和 host
var CUSTOM_PROTOCOL_SCHEME = 'https';
var QUEUE_HAS_MESSAGE = '__wvjb_queue_message__';

// 保存具体的 js 端调用 oc 之后，需要的回调方法
var responseCallbacks = {};
// 每个回调都需要的独一无二的 id
var uniqueId = 1;
```

##### window.WebViewJavascriptBridge

`WebViewJavascriptBridge_js`   还为 window 创建了一个 `WebViewJavascriptBridge` 对象。其中包含了几个方法：

```js
window.WebViewJavascriptBridge = {
  registerHandler: registerHandler,
  callHandler: callHandler,
  disableJavscriptAlertBoxSafetyTimeout: disableJavscriptAlertBoxSafetyTimeout,
  _fetchQueue: _fetchQueue,
  _handleMessageFromObjC: _handleMessageFromObjC
};
```

##### registerHandler

`registerHandler(handlerName, handler)` 用来提供给 JS 注册 hander，以供 OC 调用，注册的方法都会保存在 `messageHandlers` 字典中：

```js
function registerHandler(handlerName, handler) {
  messageHandlers[handlerName] = handler;
}
```

##### callHandler

`callHandler(handlerName, data, responseCallback)`：JS 主动调用 OC 提供的 handler 的方法。

```js
function callHandler(handlerName, data, responseCallback) {
  _doSend({ handlerName:handlerName, data:data }, responseCallback);
}

function _doSend(message, responseCallback) {
  if (responseCallback) {
    var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
    responseCallbacks[callbackId] = responseCallback;
    message['callbackId'] = callbackId;
  }
  sendMessageQueue.push(message);
  messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

把方法名和参数生成一个消息对象，放到 `sendMessageQueue` 数组中，同时加载 URL 调起原生。如果有回调的 block，那么通过 uniqueId 生成一个 callbackId，也保存到消息对象中。

##### disableJavscriptAlertBoxSafetyTimeout

```js
function disableJavscriptAlertBoxSafetyTimeout() {
	dispatchMessagesWithTimeoutSafety = false;
}
```

是否禁用异步调用。如果设置为 false。那么native 端调用 js 的话将是同步的。我们可以在下面的 `_handleMessageFromObjC` 方法中看到相关逻辑。

##### _fetchQueue

获取所有的消息队列，转为字符串返回，然后清空消息队列

```js
function _fetchQueue() {
  var messageQueueString = JSON.stringify(sendMessageQueue);
  sendMessageQueue = [];
  return messageQueueString;
}
```

##### _handleMessageFromObjC

这个方法会被 native 调用，用来处理 native 传递过来的事件

```objc
function _handleMessageFromObjC(messageJSON) {
  _dispatchMessageFromObjC(messageJSON);
}

function _dispatchMessageFromObjC(messageJSON) {
  // 判断是否是异步调用。如果是异步调用则通过 setTimeout 将方法延后调用。否则直接执行。由于 javascript 是单线程的原因，会阻塞原有 js 代码的执行。
  if (dispatchMessagesWithTimeoutSafety) {
    setTimeout(_doDispatchMessageFromObjC);
  } else {
      _doDispatchMessageFromObjC();
  }
  
  function _doDispatchMessageFromObjC() {
    var message = JSON.parse(messageJSON);
    var messageHandler;
    var responseCallback;

    if (message.responseId) {
      responseCallback = responseCallbacks[message.responseId];
      if (!responseCallback) {
        return;
      }
      responseCallback(message.responseData);
      delete responseCallbacks[message.responseId];
    } else {
      if (message.callbackId) {
        var callbackResponseId = message.callbackId;
        responseCallback = function(responseData) {
          _doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
        };
      }
      
      var handler = messageHandlers[message.handlerName];
      if (!handler) {
        console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
      } else {
        handler(message.data, responseCallback);
      }
    }
  }
}
```

解析从 native 传来的 `messageJSON`，如果存在 `responseId` 这个字段，说明是之前 js 调用 native 后的回调，那么就需要在自己的 `responseCallbacks` 找对应的回调方法，然后执行。执行完毕后通过 `delete` 删除。

如果存在 `callbackId` 字段，说明 JS 在处理好 native 的调用后，还需要回调 native 的方法。所以就先创建一个 callback 的闭包，把必要的信息都存到这个闭包中。然后在 `messageHandlers` 中找对应的处理方法，把调用的数据和回调闭包传入执行。

##### 创建发送消息的 iframe

```js
// 创建 iframe，用来加载 URL 发送消息给 Native
messagingIframe = document.createElement('iframe');
messagingIframe.style.display = 'none';
messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;  // https://__wvjb_queue_message__
document.documentElement.appendChild(messagingIframe);
```

##### 执行 bridge 创建完成的回调

到这一步，js 端的 bridge 基本创建完毕了，现在就可以使用这个 bridge 注册提供给 oc 的方法了。执行 `windows.WVJSCallbacks` 数组中保存的各个回调。

```js
setTimeout(_callWVJBCallbacks, 0);
function _callWVJBCallbacks() {
  var callbacks = window.WVJBCallbacks;
  delete window.WVJBCallbacks;
  for (var i=0; i<callbacks.length; i++) {
    callbacks[i](WebViewJavascriptBridge);
  }
}
```

这里可以看到，业务端要执行的回调方法 `WVJSCallbacks` 无法直接传给 `WebViewJavaScriptBridge_JS` ，所以就通过挂载到 window 下的方式，让后者取到。

#### JS 调用原生

JS 调用原生的方法上面已经分析过了，就是 `window.WebViewJavascriptBridge。callHandler` 方法，会来到 `decidePolicyForNavigationAction` 方法，并且进入 `WKFlushMessageQueue` 方法中：

```objc
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    if (webView != _webView) { return; }
    NSURL *url = navigationAction.request.URL;
    __strong typeof(_webViewDelegate) strongDelegate = _webViewDelegate;

    if ([_base isWebViewJavascriptBridgeURL:url]) {
        if ([_base isBridgeLoadedURL:url]) {
            [_base injectJavascriptFile];
        } else if ([_base isQueueMessageURL:url]) {
            [self WKFlushMessageQueue];
        } else {
            [_base logUnkownMessage:url];
        }
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    ...
}
```

`WKFlushMessageQueue`  会先去 js 端获取 js 端要执行的 native 的所有方法名和参数，然后执行 `_base` 中的 `flushMessageQueue` 方法：

```objc
- (void)WKFlushMessageQueue {
    [_webView evaluateJavaScript:[_base webViewJavascriptFetchQueyCommand] completionHandler:^(NSString* result, NSError* error) {
        if (error != nil) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Error when trying to fetch data from WKWebView: %@", error);
        }
        [_base flushMessageQueue:result];
    }];
}
```

`flushMessageQueue` 方法会先把从 JS 端传来的字符串转为对象数组，然后在自身注册的 `messageHandlers` 中找对应的处理方法执行。

```objc
- (void)flushMessageQueue:(NSString *)messageQueueString{
    // 把 js 端传来的 String 转为数组对象
    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
		...
        NSString* responseId = message[@"responseId"];
        if (responseId) {
            // 存在 responseId，表明是原本 OC 调用需要的 callback。在 oc 的 responseCallback 
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else {
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            // 如果有 callbackId 说明 js 端需要回调
            if (callbackId) {
                // 创建一个 callback 的 block，传入 responseData，然后找到 JS 端响应的处理方法
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    // 把需要回调的 callbackId 作为 responseId 存入
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            
            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            // 执行 handler 方法
            handler(message[@"data"], responseCallback);
        }
    }
}
```

`_queueMessage` 方法将在下面的原生调用 JS 中分析

#### 原生调用 JS

原生调用 JS，bridge 先调用 `_base` 中的相关处理方法:

```objc
// WKWebViewJavascriptBridge.m
- (void)callHandler:(NSString *)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback {
    [_base sendData:data responseCallback:responseCallback handlerName:handlerName];
}
```

`_base` 中把 `data` `callbackId` `handlerName` 这几个参数封装为字典，再转为 JSON字符串。随后在主线程中通过 `evaluteJavascript` 传递给JS：

```objc
// 参数转为字典
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    
    if (data) {
        message[@"data"] = data;
    }
    
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
    
    if (handlerName) {
        message[@"handlerName"] = handlerName;
    }
    [self _queueMessage:message];
}

- (void)_queueMessage:(WVJBMessage*)message {
    if (self.startupMessageQueue) {
        [self.startupMessageQueue addObject:message];
    } else {
        [self _dispatchMessage:message];
    }
}

// 字典转为 JSON 并执行
- (void)_dispatchMessage:(WVJBMessage*)message {
    NSString *messageJSON = [self _serializeMessage:message pretty:NO];
    [self _log:@"SEND" json:messageJSON];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    if ([[NSThread currentThread] isMainThread]) {
        [self _evaluateJavascript:javascriptCommand];

    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self _evaluateJavascript:javascriptCommand];
        });
    }
}
```

可以发现，转化后的 JSON 作为 `WebViewJavascriptBridge._handleMessageFromObjC('%@')` 的参数传入 JS。至此，原生调用 JS 的过程也结束了。

### 几个问题

1. **js 端还没有初始化完成，native 就发送的消息如何处理？**

WebViewJavaScriptBridge 提供了一个 `startupMessageQueue` 用于保存在 JS 还没有初始化完成时候的 Native 消息队列。在 JS 初始化的代码执行完后，会立即执行 `startupMessageQueue` 中保存的消息，然后把 `startupMessageQueue` 队列置位 nil，之后的消息就不会保存到 `startupMessageQueue` 中了。

2. **js 和 native 相互通信的方式是怎样？**

js 端存在一个 `sendMessageQueue` 队列，用于存放 js 需要执行的消息队列，然后通知 native 到这个消息队列中拿消息。native 端需要发送消息的时候，则是直接执行 `evaluateJavascript`。这是因为执行 js 端的消息时异步的，执行期间可能有其他消息发生，而 native 的 `evalutejavascript` 是同步的，只有执行完这个方法， native 才会继续执行。

3. **为什么`WebViewJavascriptBridge` 中 JS 调用原生时，把要传给原生的数据放到 messageQueue 中，再让原生调 JS 去取，而不是直接拼在 URL 后面？**

URL 太长会丢数据。尤其是对参数进行 base64 编码，以保证 url 中不会出现一些非法的字符的时候。如果参数是一个很复杂的对象，那么这个 url 的编解码将会很复杂。

4. **`WebViewJavascriptBridge` 中加载 URL 调起原生时，为什么不是用 `window.location="https://xxx"` 这种形式，而是新添加一个 iframe 来加载这个 URL？**

如果我们连续 2 个 js 调 native，连续 2 次改 window.location 的话，在 native 的 delegate 方法中，只能截获后面那次请求，前一次请求由于很快被替换掉，所以被忽略掉了。

5. 把 native 提供给 js 的方法都注册到 handler 中，当方法多的时候，不易于代码管理。该如何调整使不同类型的方法的职责分工更加明确？

不直接注册所有的方法，而是只注册一个方法，所有 js 调用都经过这个而方法。这个方法内部使用 runtime 动态转发实现。因此，js 端调用 native 方法的时候，需要传递类名和方法名。

## 参考

[WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)

[iOS中UIWebView与WKWebView、JavaScript与OC交互、Cookie管理看我就够](https://www.jianshu.com/p/870dba42ec15)

[从零收拾一个hybrid框架（二）-- WebView容器基础功能设计思路](http://awhisper.github.io/2018/03/06/hybrid-webcontainer/)

[自己动手打造基于 WKWebView 的混合开发框架](https://lvwenhan.com/ios/462.html)

[WebViewJavascriptBridge 源码中 Get 到的“桥梁美学”](https://juejin.im/post/5a40492f6fb9a0451969ce95#heading-7)



