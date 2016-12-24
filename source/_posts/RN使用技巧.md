title: React-Native 技巧与坑总结
date: 2016/11/25 10:07:12  
categories: React-Native
tags:
	- React-Native
	- 爬坑

---

本文将收集关于 React-Native 的各种技巧与坑，不论是摘录的还是自己遇到的。

<!--more-->






### 关于Prop
#### propTypes
组件的属性可以接受任意值，字符串、对象、函数等等都可以。有时，我们需要一种机制，验证别人使用组件时，提供的参数是否符合要求。组件类的 `PropTypes` 属性，就是用来验证组件实例的属性是否符合要求。

```javascript
class Greeting extends React.Component {
  render() {
    return (
      <Text>{this.props.name}</Text>
    );
  }
}

Greeting.propTypes = {
  name: React.PropTypes.string
};
```

上面例子中，如果 `name` 不是 string 类型，那么就会产生一个警告。还可以设置 `name: React.PropTypes.string.isRequired` 表示必须传入属性 `name`。

除了 string 外，还有许多类型的 PropTypes 可以设置。[参见](https://facebook.github.io/react/docs/typechecking-with-proptypes.html) 再举一个设置单一子节点的例子：

```javascript
class MyComponent extends React.Component {
  render() {
    // This must be exactly one element or it will warn.
    const children = this.props.children;
    return (
        {children}
    );
  }
}

MyComponent.propTypes = {
  children: React.PropTypes.element.isRequired
};
```

#### defaultProps
可以在 `defaultProps` 中注册设置默认属性值。

```javscript
class Greeting extends React.Component {
  render() {
    return (
      <Text>{this.props.name}</Text>
    );
  }
}

Greeting.defaultProps = {
  name: 'hahaha'
};
```

结合上面这两个属性，就不必再在构造函数里设置各种值了。


### this.props.children
`this.props` 对象的属性与组件的属性一一对应，但是有一个例外，就是 `this.props.children` 属性。它表示组件的所有子节点。类似于 `TouchableOpaque` 里嵌入 `Text`，通过这种方式可以很方便的嵌套封装控件。

```javascript
class NewComponent extends React.Component{
	render(){
		return(
			{this.props.children}
		);
	}
}

//调用：
<NewComponent>
	<Text>haha</Text>
</NewComponent>
```


### 关于 import，require，export，module.exports的区别
ES6标准发布后，module 成为标准，标准的使用是以 export 指令导出接口，以 import 引入模块，但是在我们一贯的 node 模块中，使用 require 引入模块，使用 module.exports 导出接口。那么在混用的情况下到底有什么异同呢？

#### export,module.exports
首先要明确一点，现今的所有 class 都会被编译器转化为同名的 function，function 内部通过 babel 提供的方法实现 class 内的各个属性和方法。

- `export xx`导出的都是 {} 括起来的对象。如果是对象，那么自动变为 `{target名:target}` 的形式；如果后面跟的是 function，那么会自动变为：`{function名:function}` 的形式。因此一般不能 export 匿名对象和方法。有一个特殊用法 `export default 对象/方法` 相当于导出了一个别名为 default 的对象/方法 `{default:对象/方法}` 一个文件内，可以使用多次 export，但只能将一个对象/方法设置为 default。
- `module.exports = xx` 较为直接，赋值的是什么，导出的就是什么。

#### import,require
- import 一般和 export 一起使用，是一种解构的方式。如：`import a {b,c} from './xx'`，等价于 `import {default as a,b,c} from './xx'`。相当于将 default 导出，并给 default取了个别名 a。
- require 一般和 module.exports 一起使用，直接将 module.exports 导出的对象赋给接收对象。如：`var a = require('./xx')`。
#### 混用
- 如果 import 和 `module.exports = xx` 一起使用，则直接 import。如:`import xx from './xx'`。
- 如果 require 和 export 一起使用则相当于将 export 生成的导出对象付给接收对象。如：`export default funcA`,导入方式一样，`var a = require('./xx');` 但要这样使用 `a.default();`


综上，两种export的方式，对于import都一样，但是对于 require有所不同。[参考链接](http://www.tuicool.com/articles/uuUVBv2)



### 关于编译时 RCTHTTPRequestHandler.m 不存在的问题

升级了一下 React-Native 的版本后（3.2=>3.8），Xcode 编译的时候出现了 `RCTHTTPRequestHandler.m not found` 的编译错误。原因应该是 `RCTHTTPRequestHandler.m` 在后面的版本变为了 `RCTHTTPRequestHandler.mm`。

解决方法很简单，在 React-Native 版本升级，即使用 `npm update` 后，需要重新 `pod install` 一遍。因为 React 作为 pod 中的一项，Xcode 是通过 `pod install` 后生成的索引来定位 React 中的各个文件的。只 `npm update` 而不 `pod install`，Xcode 编译的时候通过 3.2 的文件索引，去 3.8 里找文件，就很有可能因为版本变化导致找不到文件。 
 

### TextInput 隐藏键盘
Native 中的 `UITextField` 可以通过 `resignFirstResponder` 或者 `endEditing` 的方式取消第一响应者，从而隐藏虚拟键盘。那么，react 中如何做到隐藏键盘呢？

可以使用 `ScrollView` 包装我们的 `View`。
`ScrollView` 可以设置 `keyboardDismissMode`，`keyboardShouldPersistTaps` 来控制输入法的行为。

```jsx
<ScrollView 	contentContainerStyle={{flex:1}//非常重要，让ScrollView的子元素占满整个区域
				keyboardDismissMode='on-drag' //拖动界面输入法退出
				keyboardShouldPersistTaps={false} //点击输入法意外的区域，输入法退出
				>
....
</ScrollView>
```

### 生命周期回调函数总结
#### componentWillMount()
`componentWillMount` 会在组件 `render` 之前执行，并且永远都只执行一次。

#### componentDidMount()
`componentDidMount` 会在组件加载完毕之后立即执行。

#### componentWillReceiveProps(object nextProps)
在组件接收到一个新的 prop 时被执行。这个方法在初始化 `render` 时不会被调用。

#### boolean shouldComponentUpdate(object nextProps, object nextState)
返回一个布尔值。在组件的 props 或者 state 改变时被执行。在初始化时或者使用  `forceUpdate` 时不被执行。

如果 `shouldComponentUpdate` 返回 `false`,`render()` 则会在下一个 state change 之前被完全跳过。(另外 `componentWillUpdate` 和  `componentDidUpdate` 也不会被执行)默认情况下 `shouldComponentUpdate` 会返回 `true`.

#### componentWillUpdate(object nextProps, object nextState)
组件接收到新的 `props` 或者 `state` 但还没有 `render` 时被执行。在初始化时不会被执行。一般用在组件发生更新之前。

#### componentDidUpdate(object prevProps, object prevState)
在组件完成更新后立即执行。在初始化时不会被执行。一般会在组件完成更新后被使用。例如清除 notification 文字等操作。

#### componentWillUnmount()
主要用来执行一些必要的清理任务。**注意，`Unmount` 的大小写。**


### 如何判断对象是否有某个属性
- 使用in关键字 该方法可以判断对象的自有属性和继承来的属性是否存在。
	
	```
	var o={x:1};
	"x" in o; //true，自有属性存在
	"y" in o; //false
	"toString" in o; //true，是一个继承属性
	```
	
- 使用对象的hasOwnProperty()方法 该方法只能判断自有属性是否存在，对于继承属性会返回false。
	
	```
	var o={x:1};
	o.hasOwnProperty("x"); 　　 //true，自有属性中有x
	o.hasOwnProperty("y"); 　　 //false，自有属性中不存在y
	o.hasOwnProperty("toString"); //false，这是一个继承属性，但不是自有属性
	```
	
- 用undefined判断 自有属性和继承属性均可判断。
	
	```
	var o={x:1};
	o.x!==undefined; //true
	o.y!==undefined; //false
	o.toString!==undefined //true
	```
- 在条件语句中直接判断
	
	```
	var o={};
	if(o.x) o.x+=1; //如果x是undefine,null,false," ",0或NaN,它将保持不变
	```



### 可取消的Promise
`Promise`是 React Native 开发过程中用于异步操作的最常用的 API，但 Promise 没有提供用于取消异步操作的方法。为了实现可取消的异步操作，我们可以为 Promise 包裹一层可取消的外衣。    

```javascript
const makeCancelable = (promise) => {
  let hasCanceled_ = false;
  const wrappedPromise = new Promise((resolve, reject) => {
    promise.then((val) =>
      hasCanceled_ ? reject({isCanceled: true}) : resolve(val)
    );
    promise.catch((error) =>
      hasCanceled_ ? reject({isCanceled: true}) : reject(error)
    );
  });
  return {
    promise: wrappedPromise,
    cancel() {
      hasCanceled_ = true;
    },
  };
};  
```

然后可以这样使用取消操作：   

```javascript
const somePromise = new Promise(r => setTimeout(r, 1000));//创建一个异步操作
const cancelable = makeCancelable(somePromise);//为异步操作添加可取消的功能
cancelable
  .promise
  .then(() => console.log('resolved'))
  .catch(({isCanceled, ...error}) => console.log('isCanceled', isCanceled));
// 取消异步操作
cancelable.cancel();   
```

Promise后面括号内跟的是要异步执行的操作，`.then()`里跟的是异步操作执行完后的回调函数。取消 Promise 换句话说也就是取消 `.then()` 里的回到函数的执行。

这里通过创建一个辅助的 Promise 来包裹要执行的 Promise，这个设计非常的巧妙。

首先，如何实现取消 `.then()` 里的毁掉函数的执行？那么就不能用 `.then(callback)` 的形式了，而要有一个标识位判断：`.then(()=>flag?执行:不执行)`。

那么 `flag` 定义在哪？肯定不能直接作为一个变量定义在执行 Promise 的类里，因为这个类可能有多个 Promise 任务要处理，创建多个 `flag` 显然不是一个好的选择。那么就可以新建一个 `class`，这个 `class` 里包含 `flag` 以及这个 Promise，这就需要每次创建 Promise 的时候都 `new` 一个对象出来。这个方式可行，但是不够优雅。由于 Js 是弱类型的，我们没有必要专门定义一个 `class`，反正都是 `var`。因此，就应该像上面那样通过一个方法，直接 `return` 一个 `{}` 包裹起来的对象。因为 `return` 的这个对象用到了 `hasCanceled` 这个参数，由于有闭包性，在 `return` 的这个对象被销毁前 `hasCanceled` 都是可触及的。

最后，为什么要用一个辅助的 Promise 去包裹？ 其实用一个类或者一个方法也是能达到同样的效果。这样的设计也很巧妙。将要执行的 Promise 的 `.then()` 作为 辅助的 Promise 的异步执行操作，达到的目的是在 `.then()` 完成后，辅助的 Promise 的异步操作才可能结束。当辅助的 Promise 的异步操作结束后，就可以调用其自己的 `.then()` 来通知要执行的 Promise 已经执行完毕（仔细想了想，其实用辅助 Promise 也没甚屌用，因为 `.then()` 是可以链式调用的如：`.then().then()`，我完全可以直接自己定义一个 `function`，比如：`(callbackLogical,callbackNotify)=>promise.then(()=>flag?执行callbackLogical:不执行callbackLogical).then(callbackNotify)`。只要调用了这个方法，那么不就都搞定了么。




### AsyncStorage存储key管理小技巧   
`AsyncStorage` 是一个简单的、异步的、持久化的 `Key-Value` 存储系统，它对于 App 来说是全局性的。它用来代替 `LocalStorage`。

推荐使用封装好的第三方库[react-native-storage](https://github.com/sunnylqm/react-native-storage/blob/master/README-CHN.md) 具体使用方式就不细说了。



### 全局变量
正如 web 中的 `window`，在 RN 中也有一个全局根属性可以挂载各种全局变量：`global`。比如上面推荐的 `react-native-storage` 用到的 `storage`,使用 `global.storage = storage` 就可以在各个地方拿到 `storage` 对象。

但是要注意一点，一定要注意执行顺序，也就是必须要在使用 `global.storage` 前，将 `storage` 赋给 `global.storage`，所以要将这个赋值放到必须执行的文件中。

### 优化切换动画卡顿的问题

使用API `InteractionManager`，它的作用就是可以使本来 JS 的一些操作在动画完成之后执行，这样就可确保动画的流程性。当然这是在延迟执行为代价上来获得帧数的提高。
```javascript
	InteractionManager.runAfterInteractions(()=>{
		//...耗时较长的同步任务...
		//更新state也需要时间
		this.setState({
			...
		})
		//获取某些数据，需要长时间等待
		this.fetchData(arguements)
	})
```

### React-Native 原生模块调用(iOS)
在项目中遇到地图,拨打电话,清除缓存等iOS与Andiorid机制不同的功能,就需要调用原生的界面或模块。

#### 创建原生模块，实现“RCTBridgeModule”协议
```
#import <UIKit/UIKit.h>
#import "RCTBridgeModule.h"

@interface LoginViewController : UIViewController<RCTBridgeModule>

@end
```
#### 导出模块，导出方法
不仅可以让导出 native 的方法，而且还可以在 js 中添加回调函数，供 native 调用，这样 native 就可以将前面的数据回塞给 js 了。
```
@implementation LoginViewController
//导出模块
RCT_EXPORT_MODULE()
- (void)viewDidLoad {
    [super viewDidLoad];
}

RCT_EXPORT_METHOD(showSVProgressHUDErrorWithStatus:(NSString *)state callBack:(RCTResponseSenderBlock)callback){
  NSLog(@"state is %@",state);
  NSArray *events = [[NSArray alloc] initWithObjects:@"hello", nil];
  // 这里callback必须是数组
  callback(events);
  [SVProgressHUD showErrorWithStatus:state];
}

@end

```

#### js文件中调用
```
//创建原生模块实例
let LoginViewController = NativeModules.LoginViewController;

//方法调用
LoginViewController.showSVProgressHUDErrorWithStatus('请输入正确的手机号',(callbackString) => {console.log(callbackString);});     
```





### React Native 真机调试 
开发中真机调试是必不可少的,有些功能和问题模拟器是无法重现的,所以就需要配合真机测试

#### iOS 真机调试
首先，**必须** 保证调试用电脑的和你的设备处于相同的 `WiFi` 网络环境中下。然后修改`AppDelegate.m` 文件，设置 `jsLocation` 为本地 ip 即可。

```objc
NSURL *jsCodeLocation;
[RCTBundleURLProvider sharedSettings].jsLocation = @"192.168.31.142";
jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];

RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AwesomeProject"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
```

### React Native import 文件的小技巧
开发中经常需要 import 其他 js 文件，如果需要同时导入一些相关的 js 文件时，可以创建一个索引文件方便引用。  

#### 第一步：创建index.js   
在 `index.js` 中 import 相关的 js 文件

```
'use strict';

import Request from './network/RequestManager';
import AppContext from './network/AppContext';
import ApiServiceFactory from './network/ApiServiceFactory';

module.exports = {
    ApiServiceFactory,
    Request,
    AppContext
};
```

#### 第二步：使用   
如果需要使用这些类，只需要导入 `index` 文件就可以了~

```
import {Request, ApiServiceFactory, AppContext} from '../expand/index';
```


### React Native 读取本地的json文件  
可以以导入的形式，来读取本地的json文件，导入的文件可以作为一个js对象使用，这样方便调试的时候加载数据。

#### 导入json文件：   
```javascript
var langsData = require('./json/langs.json');
```

现在你可以操作`langsData`对象了。  

#### json格式

```json
[
  {
    "path":"",
    "name":"123",
    "checked":false
  },
  {
    "path":"aa",
    "name":"1234",
    "checked":false
  },
  {
    "path":"ddd",
    "name":"123123123",
    "checked":true
  }
]
```

#### 使用
```javascript
for (var i=0,l=langsData.length;i<l;i++){
	console.log(langsData[i]);
}
```


