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



## 参考

[WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)

[iOS中UIWebView与WKWebView、JavaScript与OC交互、Cookie管理看我就够](https://www.jianshu.com/p/870dba42ec15)

[从零收拾一个hybrid框架（二）-- WebView容器基础功能设计思路](http://awhisper.github.io/2018/03/06/hybrid-webcontainer/)

[自己动手打造基于 WKWebView 的混合开发框架](https://lvwenhan.com/ios/462.html)

[WebViewJavascriptBridge 源码中 Get 到的“桥梁美学”](https://juejin.im/post/5a40492f6fb9a0451969ce95#heading-7)



