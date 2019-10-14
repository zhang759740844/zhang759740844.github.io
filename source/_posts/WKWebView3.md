title: WKWebView 使用（三）—— WebViewJavaScriptBridge 源码解析
date: 2018/9/31 14:07:12  
categories: iOS
tags: 

- 源码解析

------

记录一下 WKWebView 相关使用

<!--more-->

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

JS 调用原生的方法上面已经分析过了，就是 `window.WebViewJavascriptBridge`的`callHandler` 方法，会来到 `decidePolicyForNavigationAction` 方法，并且进入 `WKFlushMessageQueue` 方法中：

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

5. **把 native 提供给 js 的方法都注册到 handler 中，当方法多的时候，不易于代码管理。该如何调整使不同类型的方法的职责分工更加明确？**

不直接注册所有的方法，而是只注册一个方法，所有 js 调用都经过这个而方法。这个方法内部使用 runtime 动态转发实现。因此，js 端调用 native 方法的时候，需要传递类名和方法名。