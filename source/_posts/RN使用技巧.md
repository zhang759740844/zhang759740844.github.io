title: React-Native 技巧与坑总结
date: 2016/11/25 10:07:12  
categories: React-Native
tags:
	- React-Native
	- 爬坑
	- 持续更新

---

本文将收集关于 React-Native 的各种技巧与坑，不论是摘录的还是自己遇到的。

<!--more-->

### `keyboardShouldPersistTaps` 的使用

`keyboardShouldPersistTaps` 这是 `scrollview` 中的一个属性。

那么场景会用到这个属性呢？就是在一个 `scrollview` 中有两个 `textinput` a,b，当 a 输入完之后点击 b，这个时候如果你不设置 `keyboardShouldPersistTaps` 属性，那么点击 b 后，虚拟键盘消失，你需要再点一次 b 才能将虚拟键盘再打开，也就是说 `scrollview` 并没有相应 b 控件的点击事件。正常的需求应该是，点击 b 后，键盘仍然代开状态，只不过输入框变为 b。所以要用该属性控制。

该属性有三个枚举值：

- `never`: 默认情况，点击 `TextInput` 以外的子组件会使当前的软键盘收起。此时子元素不会收到点击事件。
- `always`: 键盘不会自动收起，`ScrollView` 也不会捕捉点击事件，但子组件可以捕获。
- `handled`:当点击事件被子组件捕获时，键盘不会自动收起。这样切换 `TextInput` 时键盘可以保持状态。多数带有TextInput的情况下你应该选择此项。



### `cloneWithRows` 使用的注意事项

我们知道 `cloneWithRows` 是在 `listview` 中保存列表数组时使用的方法。使用的时候要注意一点：数组在 `cloneWithRows` 之后会变成一个特殊的数据结构，数据数组只是这个数据结构中的一个属性。

那么什么时候需要注意呢？一个父控件内的子控件里有一个 `listview`，那么要么传入数据在里面 `cloneWithRows`，要么在外面 `cloneWithRows` 好后直接传入，不能外部 `cloneWithRows` 一次后再在里面 `cloneWithRows` 一次。推荐是在外面 `cloneWithRows` 好后传入，这样更符合封装性。

### View 设置宽高

一个视图的显示必须要有宽高。有以下几种情况：

1. 父子控件都设置了绝对宽高：那么父子控件都按照自己的绝对宽高布局。子控件可能超出父控件
2. 父控件设置了绝对宽高，子控件使用 `flex`,`margin` 等：子控件按照父控件的宽高进行调整。适用于父控件要经常变换宽高的情况。
3. 子控件设置了绝对宽高，父控件无宽高设置：父控件正好包裹子控件。适用于子控件要经常变换宽高的情况。

### 设置 Image

`Image` 图片一定要设置宽高，因为如果图片默认大小是0，加载完图片后，会有个闪烁，可以设置主题的图片模式是 `resizeMode = ‘contain’`  这样图片就能在指定大小内适应缩放。

### JSX 中的一点小注意事项

在 JSX 中调用外部的 js方法，如果要用到 `this`， 则必须 `bind(this)` 或者使用箭头函数，否则无法识别。

JSX 标签里一定不能有 `{}`，就比如你要把 `View` 里的 `style` 注释掉， 一定不能直接用 `cmd+/` 这样会在 ` <>`里加入 `{}`，产生 `SyntaxError xxx.js Unexpected token,expected ...`的错误

`<View >` 标签里的属性必须要要遵守如下的形式，即必须要用等号，并且**要用 `{}` 把对象或者返回对象的方法包裹起来**:

```jsx
<View 
    style = {{margin}}>
</View>
```



### 如何隐藏一个组件

如果让一个组件隐藏，或者根据不同情况改变组件展示。只需要在必要的时候通过 `state` 的变化，将原来 `return` 的 `view` 变成 `return null` 就可以了

```jsx
_render() {
    return(
    	...
        {
  			this.state.drawerOpen ?
  			<TouchableOpacity style={styles.modalContainer} /> : null
		}
    	...
    )
}
```

**注意用 `{}` 包裹的部分，要么就像上面那样的一个二元选择或者是直接的一个 JSX 对象，要么就是下面这样的调用一个返回 JSX 的方法：**

```
_render() {
    return(
    	...
        {
			this._renderContent(name, type)
		}
    	...
    )
}

_renderContent(name, type) {
	if (type === 1) {
        return(
    		<View/>
    	)
	} else {
        return(
        	<Image/>
        )
	}
}
```

**不能直接写 js 的逻辑语句，一定会报错**

### padding 和 margin 使用区别

这两者 android 程序员使用起来恐怕没有任何问题。iOS 程序员如果使用惯了 `autoLayout` 可能一时反应不过来。

style 究竟是在父控件里用 `padding` 还是在子控件里用 `margin`。其实基本没有太大差别，一般用 `margin`，能让子控件的布局更灵活一些。当然，如果父控件的样式需要复用多次，而子控件各不相同时，直接在父控件设置 `padding`，可以减少每次设置子控件 `margin` 的次数。

### 组件之间的通信

#### 子组件调用父组件方法
父组件将方法以属性的方式传入子组件，子组件通过 `this.props.方法名` 拿到这个方法。

#### 父组件调用子组件方法
父组件调用子组件的条件是拿到子组件的实例。因此可以为子组件加上 `ref` 属性。比如：

```jsx
<Child ref='child'>haha</Child>
```

这样父组件就可以通过 `this.ref.child` 来获取 `Child` 组件的实例，并调用其内部方法了。

比较典型的用法在于一个 View 里嵌了一个 ListView，现在要调用 ListView 的刷新方法。就可以通过 `ref` 的方式从外部拿到。

### State



### Props

#### 简介

组件在创建的时候传入 `Props` 来完成定制，例如：

```jsx
<Image source={pic} style={{width: 193, height: 110}} />
```

其中 `source`,`style` 都是传入 image 的 `Props`。其中 `pic` 表示一个js对象，类似后面的 `{width: 193, height: 110}`。

`{pic}` 外面有个括号，表示括号内是一个js变量或者表达式，需要执行后取值，以此**在JSX中嵌入单条js语句**。

#### propTypes
组件的属性可以接受任意值，字符串、对象、函数等等都可以。有时，我们需要一种机制，验证别人使用组件时，提供的参数是否符合要求。组件类的 `PropTypes` 属性，就是用来验证组件实例的属性是否符合要求。

```jsx
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

```javascript
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




### TextInput 隐藏键盘
Native 中的 `UITextField` 可以通过 `resignFirstResponder` 或者 `endEditing` 的方式取消第一响应者，从而隐藏虚拟键盘。那么，react 中如何做到隐藏键盘呢？

可以使用 `ScrollView` 包装我们的 `View`。
`ScrollView` 可以设置 `keyboardDismissMode`，`keyboardShouldPersistTaps` 来控制输入法的行为。

```jsx
<ScrollView 	contentContainerStyle={{flex:1}}//非常重要，让ScrollView的子元素占满整个区域
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

这个方法很重要。组件内部属性的初始化设置只有一次，所以当组件初始化完成后，外部传入的属性值的变化不会直接引起组件内部属性值的变化，而是会回调这个方法。

> 如果你在组件内部用一个变量去接 props，那么除了在 constructor 里将 props 赋值给变量外，还需要在这个方法里将 props 赋值给变量。

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

  ```javascript
  var o={x:1};
  "x" in o; //true，自有属性存在
  "y" in o; //false
  "toString" in o; //true，是一个继承属性
  ```

- 使用对象的hasOwnProperty()方法 该方法只能判断自有属性是否存在，对于继承属性会返回false。

  ```javascript
  var o={x:1};
  o.hasOwnProperty("x"); 　　 //true，自有属性中有x
  o.hasOwnProperty("y"); 　　 //false，自有属性中不存在y
  o.hasOwnProperty("toString"); //false，这是一个继承属性，但不是自有属性
  ```

- 用undefined判断 自有属性和继承属性均可判断。

  ```javascript
  var o={x:1};
  o.x!==undefined; //true
  o.y!==undefined; //false
  o.toString!==undefined //true
  ```
  ​




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
```objective-c
#import <UIKit/UIKit.h>
#import "RCTBridgeModule.h"

@interface LoginViewController : UIViewController<RCTBridgeModule>

@end
```
#### 导出模块，导出方法
不仅可以让导出 native 的方法，而且还可以在 js 中添加回调函数，供 native 调用，这样 native 就可以将前面的数据回塞给 js 了。
```objective-c
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
```javascript
//创建原生模块实例
let LoginViewController = NativeModules.LoginViewController;

//方法调用
LoginViewController.showSVProgressHUDErrorWithStatus('请输入正确的手机号',(callbackString) => {console.log(callbackString);});     
```

### React Native 真机调试 
开发中真机调试是必不可少的,有些功能和问题模拟器是无法重现的,所以就需要配合真机测试

#### iOS 真机调试
首先，**必须** 保证调试用电脑的和你的设备处于相同的 `WiFi` 网络环境中下。然后修改`AppDelegate.m` 文件，设置 `jsLocation` 为本地 ip 即可。

```objective-c
NSURL *jsCodeLocation;
[RCTBundleURLProvider sharedSettings].jsLocation = @"192.168.31.142";
jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];

RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AwesomeProject"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
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


