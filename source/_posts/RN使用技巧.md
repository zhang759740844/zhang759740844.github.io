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

### 如何将图片字体资源自动添加到工程

RN 项目中，会用到很多第三方的组件。这些组件在 `react-native link` 的时候会作为 library 链接到主工程下。但是存在一个问题，如果组件中包含了一些图片或者字体资源，这些资源不会在 link 的过程中被主动添加到工程中。那么我们需要手动添加。这是非常麻烦的一件事。

因此，我们需要一个自动化的将资源文件添加到工程的方法：`rnpm`。`rnpm` 是一个 RN 包管理工具，现在已经被纳入到 RN 中。

#### 添加

1. 将要加入到 iOS 以及 Android 的资源文件的路径确认。比方说在 `./assets` 文件夹下。

2. 在 `package.json` 中添加配置：

   ```json
   "rnpm": {
      “assets”: ["./assets"]
   }
   ```

   这里就是资源文件放置的路径。这里是一个路径的数组，可以放任意多个路径。

3. 终端中输入 `react-native link`。通过 link 命令就把相关资源链接到工程中去了。你会看到如下提示信息：

   ![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_link.png?raw=true) 

#### 删除

没有命令可以直接删除。需要手动执行。

##### Android

安卓直接将资源文件删除即可。（我不是十分确定）

##### iOS

在 `Build Phases > Copy Bundle Resources` 中删除相关文件索引即可。



### setState 的坑

#### 坑1

`setState` 可以将控件刷新。但是这个操作不是立刻执行的而且在某个时间一并执行的。所以当你如果改变了 state 并且要用这个 state 作为参数进行网络请求的时候，不能直接使用 `setState` 给出的值，而要先将 state 改变，然后再 `setState`:

```javascript
this.state.a = '1'
this.setState({
    a: '1'
})
```

#### 坑2

一定不能在 `setState` 的时候改变 state 的原来值。否则 state 会变成意想不到的值。比如一个数组，你**不能直接在 setState 的时候往里 push值**。你可以将数组复制，然后push 好之后再 setState，或者**先设置好 state，然后再 setState。**

##### 坑3

同一个函数中的多个 `setState`  不是分别调用的，而是等到某个时刻合并执行的。所以如果 `setState` 多次设置 state 中的某个值，前面的值的设置会被后面的覆盖掉。

比如：

```javascript
setState({
    obj: {
        ...this.state.obj
        key1: value1
    }
})
setState({
    obj: {
        ...this.state.obj
        key2: value2
    }
})
```

注意，这里虽然设置的事不同的 `key1` 和 `key2`,看似没有问题。其实我们设置的是 `obj`。`key1` 被覆盖无法设置成功。

可以改成先改变 state，然后再 `setState` 刷新视图：

```javascript
this.state.obj.key1 = value1
this.state.obj.key2 = value2
setState({
    obj
})
```

当然最推荐的还是在设置 state 中同一个值时，在一起设置。 

### Text 控件

#### 对齐

设置了宽高的 Text 控件只会在左上角显示。可以使用 `text-align` 设置文字的位置，比如 center。但是显示的时候你会发现只是水平居中。你必须再使用 `line-height` 设置高度为控件高度才能够竖直居中。

#### 宽高

Text 控件的默认宽高是正好包裹文字的，因此可以不设置宽高。但是这样的 Text 是不会折行的，如果想要 Text 折行，必须添加宽度。所以**包含 Text 的控件最好不要设置 flex 来自适应，而是设置具体的宽度**。





###  `React/RCTBridgeModule.h` file not found 解决方式

这个问题出现在 RN 从 0.40 版本前升级到 0.40 版本后的情况下。0.40 前 react 的头文件都是以 `Header Search Paths` 加入的。使用 React 的组件都需要添加头文件查找路径。当组件需要使用 React 的时候，如果在组件所在的目录下没有找到，那么就会到 `Header Search Paths` 指定的路径中查找：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_0.39.png?raw=true)

这样带来一个问题，就是每一个第三方的组件都需要设置一下 `Header Search Paths` 的路径。使用者会非常不方便。

所以 FB 想要把 React 头文件的链接统一起来。于是就有了 0.40 版本的变化：通过 `Edit Scheme` 然后添加 React 这个 Target，然后取消 `Parallelize Build`。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_Build.png?raw=true)

这样，在编译项目之前，就先把 React 编译好了。也就不再需要在 Header Search Paths 中设置了。

这样带来的改变就是原来引入头文件是相当于头文件在项目中了，使用：

```objective-c
#import "RCTBridgeModule.h"
```

现在引入头文件是引入的外部的头文件，所以要使用尖括号：

```objective-c
#import <React/RCTBridgeModule.h>
```



### `keyboardShouldPersistTaps` 的使用

`keyboardShouldPersistTaps` 这是 `scrollview` 中的一个属性。

那么场景会用到这个属性呢？就是在一个 `scrollview` 中有两个 `textinput` a,b，当 a 输入完之后点击 b，这个时候如果你不设置 `keyboardShouldPersistTaps` 属性，那么点击 b 后，虚拟键盘消失，你需要再点一次 b 才能将虚拟键盘再打开，也就是说 `scrollview` 并没有相应 b 控件的点击事件。正常的需求应该是，点击 b 后，键盘仍然代开状态，只不过输入框变为 b。所以要用该属性控制。

该属性有三个枚举值：

- `never`: 默认情况，点击 `TextInput` 以外的子组件会使当前的软键盘收起。此时子元素不会收到点击事件。
- `always`: 键盘不会自动收起，`ScrollView` 也不会捕捉点击事件，但子组件可以捕获。
- `handled`:当点击事件被子组件捕获时，键盘不会自动收起。这样切换 `TextInput` 时键盘可以保持状态。多数带有TextInput的情况下你应该选择此项。



### ListView 使用的问题

#### ListView 的宽高

ListView 在父视图的 flex 方向上默认是铺满的。flex 方向上设置的宽或者高是无效的，非 flex 方向上设置的宽或者高是有效的。

所以最好在 ListView 外面套一个 View，保证 ListView 填充满整个父 View

#### `renderRow` 方法的坑

renderRow 方法的几个参数为，`rowData`, `sectionID`, `rowID` 这几个值为**字符串类型**。 所以根据 id 采取不同行为的时候，要把 id 转化为 number 再比较。或者不要用 `===` ，否则肯定返回的是 false。

#### 初始渲染数量

ListView 为了保证渲染的性能，在最开始的时候只会部分渲染，所以需要设置 `initialListSize` 属性，设置首屏的渲染个数。否则很可能首屏需要加载的元素很多，但是实际渲染出来的元素很少。

#### `cloneWithRows` 使用的注意事项

我们知道 `cloneWithRows` 是在 `listview` 中保存列表数组时使用的方法。使用的时候要注意一点：数组在 `cloneWithRows` 之后会变成一个特殊的数据结构，数据数组只是这个数据结构中的一个属性。

那么什么时候需要注意呢？一个父控件内的子控件里有一个 `listview`，那么要么传入数据在里面 `cloneWithRows`，要么在外面 `cloneWithRows` 好后直接传入，不能外部 `cloneWithRows` 一次后再在里面 `cloneWithRows` 一次。推荐是在外面 `cloneWithRows` 好后传入，这样更符合封装性。

### View 设置宽高

**一个视图的显示必须要有宽高**。以下是几个注意点：

- 父控件设置绝对宽高，子控件也设置了绝对宽高。这种情况是最简单的情况，注意子控件可能会超过父控件。


- 父视图设置了绝对宽高，子控件是 flex 布局：
  - flex-direction 方向上的子视图必须设置明确的长度，即**主轴必须明确设置多长。否则就默认为 0**
  - 非 flex-direction 方向上的子视图默认为填充满父视图，即 **`align-items` 默认为 `sketch`**。所以这里有一个坑点，**如果你在父视图中设置 `align-items: center` 居中对齐，那么就取消了默认填充满父视图次轴这一设定，子视图一定要设置次轴上相应的长度，否则就是 0，显示不出了。** 
- 子控件设置了绝对宽高，父控件可以不设置宽高，父控件正好包裹子控件。
- 父子控件都是 flex 布局，都没有设置宽高。那么这种情况下很有可能显示不出。因为父控件需要一个 `flex-direction` 方向上的长度，但是并不能通过子控件推测出。那么就会不显示了。

### 布局方式

一般布局如果是一个给定的布局，使用 flex 布局非常的直观。但是如果布局中的元素个数会变化的时候，就需要考虑一下了。比如下面这个图。中间的部分可能按情况不同会有增减：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/flex_3.png?raw=true)

一般有两种方式：

1. 考虑父级 `align-items` 设置为 sketch，块1此时和父级一样高。这个然后设置块1，`flex-direction` 为 space-between。这样块1的子级就会均匀的分布在块1内了，不论有多少元素。所以你需要设置第一个子元素和最后一个子元素相对于块1的上边和下边的距离。

   ![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/flex_1.png?raw=true)

2. 考虑父级 `align-items` 设置为 center，块1的高度取决于内部元素的高度，块1内的元素会居中。所以需要设置子元素之间的距离。

   ![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/flex_2.png?raw=true)

**一般来说用 center 会比较好一些**

### 设置 Image

`Image` 图片一定要设置宽高，因为如果图片默认大小是0，加载完图片后，会有个闪烁，可以设置主题的图片模式是 `resizeMode = ‘contain’`  这样图片就能在指定大小内适应缩放。

### 使用 JSX

#### JSX 中的 this

在 JSX 中调用外部的 js方法，如果要用到 `this`， 则必须 `bind(this)` 或者使用箭头函数，否则无法识别。

JSX 标签里一定不能有 `{}`，就比如你要把 `View` 里的 `style` 注释掉， 一定不能直接用 `cmd+/` 这样会在 ` <>`里加入 `{}`，产生 `SyntaxError xxx.js Unexpected token,expected ...`的错误

`<View >` 标签里的属性必须要要遵守如下的形式，即必须要用等号，并且**要用 `{}` 把对象或者返回对象的方法包裹起来**:

```jsx
<View 
    style = {{margin}}>
</View>
```

#### JSX 中的代码

JSX 中可以通过 `{}` 插入代码。但是你**不能直接把代码写在里面**。`{}` 内允许你**调用一个返回 JSX 节点或者以 JSX 节点为元素的数组的方法**。

这里所说的方法可以是一个外部的方法，或者是一个三元运算符，或者是一个生成数组的方法，如 `map` 等。 

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

state 中存放一些与视图有关的变量。与视图无关的变量，直接在构造器里作为自身属性创建。可以有两种方式便便 state：

```javascript
// 方式一
this.state.someProp = 1
// 方式二
this.setState({ someProp : 1 })
```

其中，方式二能在改变 State 的同时刷新视图。

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

 `this.props.children` 是一个数组，子节点作为数组元素传入。


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

一般这个方法都放在 `componentDidMount` 里。

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

### React Native 调试 
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

#### VSCode 调试

VSCode 提供了 React-Native-Tool 来进行再 VSCode 中的调试。

```json
        {
            "name": "Debug iOS",
            "program": "${workspaceRoot}/.vscode/launchReactNative.js",
            "type": "reactnative",
            "request": "launch",
            "platform": "ios",
            "sourceMaps": true,
            "outDir": "${workspaceRoot}/.vscode/.react",
            "runArguments": [
                "--scheme",
                "CRM_DEV"
            ]
        },
```

其实执行了以下命令启动 RN：

```shell
react-native run-ios --scheme CRM_DEV 
```

但是 RN 目前有一个 BUG，使用这个命令会编译指定 scheme 对应的 target，但是安装的还是主工程名的 app：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_VSCode.png?raw=true)

每次编译都是编译为 CRM_DEV 但是安装的都是 CRM。**所以你得到这个目录下，把 CRM_DEV.app 改成 CRM.app 。 然后关掉模拟器，重新启动 VSCode 调试。**

#### Chrome 调试

Chrome 调试比较麻烦，在打开 remote debug 后，你可以在 chrome 中使用 **cmd+o** 来快速打开文件。

Chrome 比 VSCode 调试的更精确，所有断点都会走。但是 Chrome 调试比较麻烦。

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


