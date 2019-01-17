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

### 获取控件的frame

给控件添加 `onLayout` 方法回调

```js
<TouchableOpacity
  onLayout={(e) => this.rowlayouts= e.nativeEvent.layout}
/>
```

就可以通过 `this.rowlayouts` 拿到组件的宽高以及起始坐标



### `pointerEvents="none"` 的使用时机

`pointerEvents="none"` 可以让设置的视图不响应点击事件。实践下来有两个应用场景：

1. 比如有一个绝对布局的视图盖住了下面的视图，在 iOS 中可以直接将其 userInteraction 置为 false。RN 中相应的属性就是 `pointerEvents`
2. 监听手势事件的时候，我们希望获取当前手势相对于父视图的位置，即 `event.nativeEvent.locationY` ，但是如果父视图中有子视图，并且手势作用的起始点在子视图上，那么 `event.nativeEvent.locationY` 是以子视图为参照的，影响我们对于坐标的计算。这时候就可以把它的子视图设置 `pointerEvents="none"`

###存在手势的视图中的按钮点击事件不响应

这个问题主要是因为点击按钮的时候手指有略微的滑动，因此，点击事件被识别为了 Move 事件，进而相应手势而不相应按钮了。

做法是在细微移动的时候不让手势相应：

```js
onMoveShouldSetPanResponder (event, gestureState) {
  let touchCapture = Math.abs(gestureState.dx) > 5 || Math.abs(gestureState.dy) > 5
  return touchCapture
}
```



### FlatList 的性能优化

FlatList 的 `data` 中接收一个数组，作为渲染的数据源。有时候我们点击了 cell，要改变状态引起重绘，但是直接修改 `data` 中某一项的值，并不会引起重绘。因为 React 中是判断 data 的地址是否变化，显然，`data` 并没有变化地址。

如果我们 `data = data.map(item => item)` 这样新建一个数据源就有点费事了。FlatList 提供了一个属性 `extraData`，当需要重绘的时候，改变 `extraData` 的值就行。

我们可以将 `extraData` 等于某一个 state 中的布尔值，每次需要重绘的时候，让这个布尔值取反。这样 FlatList 发现这个布尔值改变了，就会引发整个 FlatList 的重绘。

整个 FlatList 的重绘也是会引发性能问题的。我们要做的是自定义一个 Cell 类，然后将数据源的某个 item 设置为一个新的对象：`Object.assign({}, item, {someChange: xxx})`这样在 FlatList 重绘触发每一个 cell renderItem 的时候，大部分 Cell 并没有收到新的 Props 就会阻止整个 cell 的重绘。只有数据变化的 cell 才会触发重绘

### 组件在动画下的隐藏和显示

组件从隐藏到显示的动画是非常好想到的，就是设置组件的 `opacity` 透明度。

而从显示到隐藏的过程就有点坑了。如果还是设置透明度为 0，那么节点还在，无法响应后面视图的点击事件。但是如果 render 的时候返回 null，那么组件节点就直接消失了，无法达到动画的效果。

这里有两种方式设置隐藏：

#### display

这里我们应该借助一个 CSS 属性 `display` ,当它为 `none` 的时候，节点还在，但是组件从视图上移除了，不会影响后面的点击事件。

这里举一个显示头部视图的例子。示例代码如下：

```js
  renderHeader () {
    if (this.props.showHeader) {
      this.state.showHeader = true
      Animated.timing(this.headerViewAlpha, {
        toValue: 1,
        duration: 700
      }).start()
    } else {
      Animated.timing(this.headerViewAlpha, {
        toValue: 0,
        duration: 700
      }).start(() => {
        this.setState({
          showHeader: false
        })
      })
    }

    // 外部让显示的情况下，内部也让显示的时候才显示
    let display = !this.state.showHeader && !this.props.showHeader ? 'none' : 'flex'
    return (
      <Animated.View style={{opacity: this.headerViewAlpha, marginBottom: -20, display}}>
        {this.props.headerView()}
      </Animated.View>
    )
  }
```

这个组件通过外部传进来的 `showHeader` 控制显示和隐藏。当外部传来的 `showHeader` 为 true 时，直接通过透明度的动画显示组件。当外部传来的 `showHeader` 为 false 时，先通过透明度动画隐藏组件，然后设置内部的 `showHeader` 为 false，重绘视图。重绘的时候因为内外部的 `showHeader` 都为 false，就将 `display` 设为 `none` 了，从而达到在动画完成后隐藏视图的效果。

> 这种方式在 render 方法中又设置了 state 其实是不太推荐的。

#### height

另外一种方式就是设置视图的高度，使其高度为 0:

```js
if (showCurrentHeader) {
  Animated.timing(this.headerViewHeight, {
    toValue: deviceHeight,
    duration: 0
  }).start(() => {
    Animated.timing(this.headerViewAlpha, {
      toValue: 1,
      duration: 700
    }).start()
  })
} else {
  Animated.timing(this.headerViewAlpha, {
    toValue: 0,
    duration: 700
  }).start(() => {
    Animated.timing(this.headerViewHeight, {
      toValue: 0,
      duration: 0
    }).start()
  })
}

renderHeader () {
  return (
    <Animated.View style={{opacity: this.headerViewAlpha, marginBottom: -20, height: this.headerViewHeight}}>
      {this.props.headerView()}
    </Animated.View>
  )
}
```

这里动态修改了 `headerViewHeight` 这个高度。不过要注意，这里修改了高度并不是说子视图就一定会隐藏的。有些组件可能还是会展示出来，这个时候就要设置 style  的 `overflow: 'hidden'`

### 带有 Gesture 的父组件会 block 子组件中 TouchableOpacity 的点击事件

把按钮放到有 Move 手势的父组件的时候会经常因为响应了父组件的 Move 手势而不响应按钮的点击事件。这个时候需要给 Move 手势的添加行为做一个限制：

```js
onMoveShouldSetPanResponder: (evt, gestureState) => Math.abs(gestureState.dx) > 5 || Math.abs(gestureState.dy) > 5
```

当移动的距离超过5之后，才响应滚动事件。

### 使用绝对路径替代相对路径

如果在 import 的时候使用相对路径，那么有些层级较深的时候会非常难看，可以使用devdependency 中安装 npm 插件：`babel-plugin-root-import`

然后在*.babelrc* 中加入：

```json
{
  "plugins": [
    [
      "babel-plugin-root-import", {
        "rootPathSuffix": "src",
        "rootPathPrefix": "@"
      }
    ]
  ]
}
```

这样就可以使用 `@` 来代替目录 `src`.

### 消除 console.log

我们可以通过 babel 插件将 console 去除。

首先在 devdependency 中安装 npm 插件：`babel-plugin-transform-remove-console`

然后在更目录下新建一个名为 *.babelrc* 的文件，在其中加入：

```json
{
  "env": {
    "production": {
      "plugins": ["transform-remove-console"]
    }
  }
}
```



### 为 FlatList 设置 ListEmptyComponent

如果直接设置 `ListEmptyComponent` 占位符你会发现，即使将 `ListEmptyComponent` 的 style 设置为 `{flex:1}` 它也并不会填充满 flatList。这是因为包裹它的外层 `View` 没有设置高度。这就需要我们自己将 `FlatList` 的高度设置给 `ListEmptyComponent`。可以使用 `onLayout` 方法：

```js
<FlatList
  onLayout={e => {
    this.setState({
      fHeight: e.nativeEvent.layout.height
    })
  }}
/>
```

我们在 `FlatList` 布局的时候获取到它的高度设置为 state 即可。



### husky hook git commit

我们可以使用 husky hook git 的提交方法。安装方式如下：

```bash
npm install husky --save-dev
```

然后在 package.json 中配置：

```json
{  
  "scripts": {
    "lint": "standard --verbose | snazzy",
  },
  "husky": {
    "hooks": {
      "pre-commit": "npm run lint",
    }
  }
}
```

需要注意的是，husky 安装的时候会在 `.git` 文件夹下生成 `hook` 文件夹，如果是拷贝的别人的 node_modules，不会生成 hook 文件夹，所以需要先 uninstall 再 install 一遍



### 清除 RN package 缓存

一般情况下我们不需要清除 package 的缓存，清除缓存后再打开会非常耗时。但是我遇到了一种情况就是使用 `babel-plugin-root-import` 使用 `@` 替代 `src` 目录的时候，`@` 的指向总是不对。所以我怀疑可能是缓存的问题。

清除缓存使用命令：

```bash
npm start -- --reset-cache
```

注意中间要加上 `--`

### Immutable.js 的使用动机

一般说到 React 性能优化就会提到 Immutable 这个库。这里介绍一下它的使用场景。

当一个页面的状态树很大的时候，我们更新叶节点的时候只希望更新与之相关的节点，比如下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/immutable_1.png?raw=true)

如果改变了其他节点的引用，可能会导致用到其他节点数据的视图重绘，浪费性能。一般情况下，当节点层数较浅的时候，我们会使用展开运算符，只改变响应节点的值，其余节点都使用原来的引用。但是节点层数深的时候，也会变得很麻烦。所以就可以使用 Immutable.js 这个库。当你改变某个叶节点的值的时候，它会自动将其相关的根节点更新为新的引用，而其他无关的节点还保持原有引用。

当然，Immutable 提供的类型毕竟不是原生类型，使用起来需要注意一些地方，所以最好还是把状态树设计的扁平一些。

### react-native link

`react-native link` 方法可以把 node_module 中的 RN 库链接到 iOS 以及 Android 工程中。但是其中有一个坑点，就是在 iOS 端，`react-native link` 只会将 RN 库链接到 default target 中，而自己另外新建的 target 需要自己动手链接到 `Link Binary with Libraries` 中

### 黄色警告

黄色警告以及红屏报错可以手动触发：

```javascript
console.error('红屏错误')
console.warn('黄屏警告')
```

我们开发的时候，应该尽可能避免黄色警告。但是如果这个警告是第三方引入的呢？我们可以隐藏特别类型的警告，比如 ant-design 引入的如下警告：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/yellowbox.png?raw=true)

我们可以通过下面的代码隐藏：

```javascript
import {YellowBox} from 'react-native'

const ignoreCase = [
  // ant design 引入的
  'Warning: NativeButton: prop type `background` is invalid;'
]
YellowBox.ignoreWarnings(ignoreCase)
```



### 何时重绘

触发重绘有两种方式：

- `setState` 调用的时候。
- `props` 变化的时候

> 现有的例子开看， `setState` 只是用来标记重绘的，标记了重绘后。React 的 `render` 方法生成的新的 JSX 对象和老的 JSX 对象比较，看看两个 JSX 对象的各个部分有哪些地方不同。然后渲染不同的部分

由于 `componentWillReceiveProps` 后面接着的就是 `render` 方法，所以 `componentWillReceiveProps` 中不需要使用 `this.setState` 。直接修改 this 上的属性也是可以的比如：

```javascript
 componentWillReceiveProps (nextProps) {
    this.count = nextProps.count
 }
 render () {
   return (
     <View>
       <Text>{this.count}</Text>
     </View>
   )
 }
```

这样也是可以正确渲染出 `this.count` 的。

###  性能优化

#### 利用 shouldComponentUpdate

针对有子组件的视图，每次父组件 `render` 的时候，都会触发子组件的 `componentWillReceiveProps` 和 `render` 方法。所以我们创建子组件的时候，最好重写 `shouldComponentUpdate` 方法，去判断 props 中的各个属性是否变化。一般出于性能原因，`shouldComponentUpdate` 方法都是进行**浅层判等**，即判断之前的属性和现在的属性是否是同一个对象:

```javascript
shouldComponentUpdate(nextProps, nextState) {
    return (nextProps.completed !== this.props.completed) || (nextProps.text !== this.props.text)
}
```

#### style 不要写在 JSX 中

这里有一个注意点，如下代码即使重写了 `shouldComponentUpadate` 方法也是一直会重绘的：

```javascript
<Foo style={{color: 'red'}} />
```

这是因为，`{color: 'red'} ` 相当于每次都传入了一个新的对象。所以传 style的时候，不要直接写在 JSX 中

> 其实任何属性，包括传一个方法都不应该直接写在 jsx 中，如果都不写在 jsx 中，就会产生很多冗余代码。所以注意 style 写在 styleSheet 中这点即可。

#### render 时不要使用箭头函数

我们在 render 一个 button 的时候经常这么写：

```jsx
<Button onClick={()=> this.doClick()}>
</Button>
```

这样会导致组件的重绘。因为每次渲染的时候会重新创建这个箭头函数，导致传入了新的 props。正确的做法应该是把这一过程提前：

```js
doClick = () => {
}
```

```jsx
<Button onClick={this.doClick()}>
</Button>
```

这样 `doClick` 方法传递的就是一个引用了。（如果有些方法需要参数，那么把参数作为 props 传入）

> 并不是说使用箭头函数一定会产生重绘，有些组件内部会重写 shouldComponentUpdate 方法，会无视这种 onClick 事件。但是如果组件内部没有这么做。就需要自己注意箭头函数引起的重绘了。

#### 使用 react-redux 的 connect 方法

我们写组件的时候写 `shouldComponentUpdate` 判断一个个 props 是否变化其实是一个蛮烦的事。react-redux 其实帮我们做了这件事。使用它提供的 `connect` 方法，不需要做任何其他的事情，只要在 export 组件的时候做一些改变即可：

```javascript
export default TodoItem
=> 改为
export default connect()(TodoItem)
```

`connect()` 参数为空，表示不从 store 中获取任何状态与方法 

#### key

key 是一个老生常谈的东西。对于一个列表的每一项，需要唯一的 key 值。新老列表，key 值不同的项，视图将会被 Unmount 以及 mount，对于 key 值相同的项，视图只会被更新。

对于一个列表，如果我们不设置 key 值，默认是使用列表数组的索引 index 作为 key。但是这样会产生性能问题。比如删除了列表的第一项，整个列表的每一项都会更新

那如果我们在列表中添加一项的时候，什么值能作为这个唯一的 key 呢？可以依靠时间：`Data.now()`。

比方说在 add 的时候，为添加的项创建一个 key 字段：

```javascript
addItem: function(e) {
  var itemArray = this.state.items;

  itemArray.push(
    {
      text: this._inputElement.value,
      key: Date.now()
    }
  );

  this.setState({
    items: itemArray
  });
}
```

这样每次添加的时候，key 就获得了唯一值。

### Reselect

使用 react-redux 的时候，还经常搭配另一个常用的库 Reselect。我们存在 redux 中的 state 可能需要经过一些处理。

比如 `state.a` 和 `state.b` 可能通过 `g(a,b)` 衍生出 c。这个 c 如果放在 redux 中，那么每个 `state.a` 和 `state.b` 变化的地方都要计算 `g(a,b)`，很容易遗漏出错。如果把 c 放在 render 方法中，即每次 render 的时候计算 `g(a,b)`，又会造成重复计算。

因此，比较好的做法就是在 `state.a` `state.b` 变化的时候计算 `g(a,b)`，并且即不把`g(a,b)` 放在 redux 中，也不放在 render 中。所以，我们可以通过 redux 把数据传给组件的时候添加计算属性的方式来达到目的，即通过  `mapStateToProps` 方法。

> 是不是很像 vuex 中的 getter。Vue 真是太人性化了

 ```javascript
import { createSelector } from 'reselect'

fSelector = createSelector(
    [state => state.a,
    state => state.b],
    (a, b) => f(a, b)
)
hSelector = createSelector(
    [state => state.b,
    state => state.c],
    (b, c) => h(b, c)
)
gSelector =  createSelector(
    [state => state.a,
    state => state.c],
    (a, c) => g(a, c)
)
uSelector = createSelector(
    [state => state.a,
    state => state.b,
    state => state.c],
    (a, b, c) => u(a, b, c)
)

...
function mapStateToProps(state) {
    const { a, b, c } = state
    return {
        a,
        b,
        c,
        fab: fSelector(state),
        hbc: hSelector(state),
        gac: gSelector(state),
        uabc: uSelector(state)
    }
}
 ```

比如上面的例子，`fab` 是通过 `ab` 计算得到，通过 `createSelector`方法，注册了 `ab`，以及计算方法 `f(a,b)`。那么只有在  `a || b` 变化的时候，才会重新计算 `fab`

### setTimeout

比较简单的一个 js 的方法，但是要记住，在某个组件被卸载（unmount）之后，计时器却仍然在运行，要解决这个问题，只需铭记`在unmount组件时清除（clearTimeout/clearInterval）所有用到的定时器`：

```javascript
import React,{
  Component
} from 'react';

export default class Hello extends Component {
  componentDidMount() {
    this.timer = setTimeout(
      () => { console.log('把一个定时器的引用挂在this上'); },
      500
    );
  }
  componentWillUnmount() {
    // 请注意Un"m"ount的m是小写

    // 如果存在this.timer，则使用clearTimeout清空。
    // 如果你使用多个timer，那么用多个变量，或者用个数组来保存引用，然后逐个clear
    this.timer && clearTimeout(this.timer);
  }
};
```



### 主线程渲染

iOS 中渲染视图要在主线程中，所以 RN 中要调用原生方法，并且渲染视图的时候要通过 `dispatch_async` 到主线程进行。

比如 present 一个页面的时候就要在主线程中，否则 `[self.retryBtn setTitle:@"重试" forState:UIControlStateNormal];` 这种设置按钮 title 的方法在非主线程中执行就无法渲染出 button 的 title

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
    a: this.state.a
})
```

#### 坑2

一定不能在 `setState` 的时候改变 state 的原来值。否则 state 会变成意想不到的值。比如一个数组，你**不能直接在 setState 的时候往里 push值**。你可以将数组复制，然后push 好之后再 setState，或者**先设置好 state，然后再 setState。**

#### 坑3

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

注意，这里虽然设置的是不同的 `key1` 和 `key2`,看似没有问题。其实我们设置的是 `obj`。`key1` 被覆盖无法设置成功。

可以改成先改变 state，然后再 `setState` 刷新视图：

```javascript
this.state.obj.key1 = value1
this.state.obj.key2 = value2
setState({
    obj
})
```

当然最推荐的还是在设置 state 中同一个值时，在一起设置。 

#### setState 原理

在React中，**如果是由React引发的事件处理（比如通过onClick引发的事件处理），调用setState不会同步更新this.state，除此之外的setState调用会同步执行this.state**。所谓“除此之外”，指的是绕过React通过addEventListener直接添加的事件处理函数，还有通过setTimeout/setInterval产生的异步调用。

为什么会这样？

在React的 `setState` 函数实现中，会根据一个变量  **isBatchingUpdates** 判断是直接更新`this.state` 还是放到队列中回头再说，而**isBatchingUpdates**默认是 false，也就表示 `setState` 会同步更新 `this.state`，但是，有一个函数**batchedUpdates**，这个函数会把**isBatchingUpdates**修改为 true，而当React在调用事件处理函数之前就会调用这个**batchedUpdates**，造成的后果，就是由React控制的事件处理过程 `setState` 不会同步更新 `this.state`。

### Text 控件

#### 对齐

设置了宽高的 Text 控件只会在左上角显示。可以使用 `text-align` 设置文字的位置，比如 center。但是显示的时候你会发现只是水平居中。你必须再使用 `line-height` 设置高度为控件高度才能够竖直居中。

> 对于 text 控件，设置 height 是没用的，默认是 text 的高度。必须要设置 `line-height` 

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

另外，如果拿一个突破作为背景的时候，一定要同时设置 `Image` 的宽高，以及 `Image` 包含的 View 的宽高。注意，**包含的 View 不会自动填充满 `Image`**。

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

这样父组件就可以通过 `this.refs.child` 来获取 `Child` 组件的实例，并调用其内部方法了。

比较典型的用法在于一个 View 里嵌了一个 ListView，现在要调用 ListView 的刷新方法。就可以通过 `ref` 的方式从外部拿到。

##### ref 属性

上面演示的是 `ref` 接受一个字符串的使用方式，**`ref` 属性还可以是一个回调函数，这个回调函数会在组件被挂载后立即执行**。被引用的组件会被作为参数传递，回调函数可以用立即使用这个组件，或者保存引用以后使用：

```javascript
  render () {
    return <TextInput ref={(element) => this._input = element} />;
  },
  componentDidMount () {
    this._input.focus();
  },
```

上面的例子中，在 `ref` 回调方法中把节点 TextInput 保存为 `_input` 属性。可以在必要的时候调用。

>我认为最好还是用回调函数，通过回调函数可以把要使用的组件提前声明出来，方便调用。

#### 跨级组件通信

跨级组件，如果还是一层层传递 props 非常的不优雅。React 提供了一个 context 属性。不过这并不推荐使用。一般我们使用 `react-redux` 库的时候，`store` 就是通过 `context` 传递的。

#### 没有嵌套关系的组件通信

没有嵌套关系可以使用 `EventEmitter` 实现。在一个地方注册，另一个地方监听。不过也是不推荐的。所以用法也就不细说了。

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

#### 子组件内获取 props

有时候，我们想要封装一个组件，在容器组件内多定义几个 props，但是并不希望这些 props 传到子组件内，比如容器组件的 children 属性。我们可以这样做：

```javascript
render () {
    const {
		style,
		children,
		...restProps,
	} = this.props;
    // 删除多余属性
    [
      'onOpenChange',
      'onDrawerOpen',
      'onDrawerClose',
      'drawerPosition',
      'renderNavigationView',
    ].forEach(prop => {
      if (restProps.hasOwnProperty(prop)) {
        delete restProps[prop];
      }
    });
    
    return (
        <View style={style}>
    		<SomeView {...restProps}/>
        	{...children}
    	</View>
    )
}

```

注意：

1. 通过对象展开符，可以获取到 props 中剩余的属性。
2. 将一个对象作为组件的属性传入的时候要通过 `{...obj}` 的方式
3. 通过 `hasOwnProperty` 进一步删除不想传递给子组件的属性
4. `this.props` 的展开要放在 `render` 方法里，因为 props 可能会变化触发重绘，所以要每次重绘的时候都进行对象展开

#### Props 使用的注意点

通常我们直接会把 props 放到 render 方法中，比如上面的例子。但是这样其实不太好，比如一个页面跳转的时候，会带一些 props 过来，我们需要修改 props 中的一些属性。但是我们并不希望把这些修改带回到其他页面。

这种时候我们就不能直接修改 props 中的属性了。我们需要在 render 的时候，深拷贝或者不浅不深的拷贝 props 的值：

```javascript
render () {
    this.props1 = this.props.props1
    this.props2 = this.props.props2
    return (
    	<View/>
    )
}
```

> 因为多加了一层 `this.props1` 我们就不需要担心，到底能不能修改 props 了，如果不能修改 props，那么直接深拷贝一下即可。

更进一步，其实我们只有在不希望修改数据带到其他页面的时候才会使用 `this` 挂载，一般情况下，我们直接使用结构赋值即可：

```javascript
render () {
    const {prop1, prop2} = this.props
    return (
    	<View/>
    )
}
```

如果项目变化需求变化了，再转到把 props 的属性挂在到 `this` 下：

```javascript
render () {
    this.props1 = this.props.props1
    this.props2 = this.props.props2
    const {prop1, prop2} = this
    return (
    	<View/>
    )
}
```

就不需要再更换 View 里的参数了

#### propTypes
组件的属性可以接受任意值，字符串、对象、函数等等都可以。有时，我们需要一种机制，验证别人使用组件时，提供的参数是否符合要求。组件类的 `PropTypes` 属性，就是用来验证组件实例的属性是否符合要求。我们需要引入一个 `prop-types` 库：

```jsx
import PropTypes from 'prop-types';

class Greeting extends React.Component {
  render() {
    return (
      <Text>{this.props.name}</Text>
    );
  }
}

Greeting.propTypes = {
  name: PropTypes.string
};
```

上面例子中，如果 `name` 不是 string 类型，那么就会产生一个警告。还可以设置 `name: PropTypes.string.isRequired` 表示必须传入属性 `name`。

除了 string 外，还有许多类型的 PropTypes 可以设置:

```javascript
MyComponent.propTypes = {
  // 可以声明prop是特定的JS基本类型
  // 默认情况下这些prop都是可选的
  optionalArray:PropTypes.array,
  optionalBool: PropTypes.bool,
  optionalFunc: PropTypes.func,
  optionalNumber: PropTypes.number,
  optionalObject: PropTypes.object,
  optionalString: PropTypes.string,
  optionalSymbol: PropTypes.symbol,

  // 任何可以被渲染的事物：numbers, strings, elements or an array
  // (or fragment) containing these types.
  optionalNode: PropTypes.node,

  // A React element.
  optionalElement: PropTypes.element,

  // 声明一个prop是某个类的实例，用到了JS的instanceof运算符
  optionalMessage: PropTypes.instanceOf(Message),

  // 用enum来限制prop只接受特定的值
  optionalEnum: PropTypes.oneOf(['News', 'Photos']),

  // 指定的多个对象类型中的一个
  optionalUnion: PropTypes.oneOfType([
    PropTypes.string,
    PropTypes.number,
    PropTypes.instanceOf(Message)
  ]),

  // 指定类型组成的数组
  optionalArrayOf: PropTypes.arrayOf(PropTypes.number),

  // 指定类型的属性构成的对象
  optionalObjectOf: PropTypes.objectOf(PropTypes.number),

  // 一个指定形式的对象
  optionalObjectWithShape: PropTypes.shape({
    color: PropTypes.string,
    fontSize: PropTypes.number
  }),

  // 你可以用以上任何验证器链接‘isRequired’，来确保prop不为空
  requiredFunc: PropTypes.func.isRequired,

  // 不可空的任意类型
  requiredAny: PropTypes.any.isRequired,
  // PropTypes.element指定仅可以将单一子元素作为子节点传递给组件
  children: PropTypes.element.isRequired
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
主要用来执行一些必要的清理任务。**注意，`Unmount` 的大小写。**写错就无法调用了！！！



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
首先，**必须** 保证调试用电脑的和你的设备处于相同的 `WiFi` 网络环境中下。然后修改`AppDelegate.m` 文件，设置 ip 为电脑 ip 即可。

然后就可以通过 Chrome 开发工具进行调试。最好不要使用 VSCode 提供的测试工具。不好用。

如果想要快速调样式，建议选上 `Enable Hot Reloading` 。可以在你每次保存的时候在本页面重新加载。

使用 xcode run 一遍之后，如果没有 native 代码的改动，手机就可以不用再连着电脑了，在项目地址下，使用 `npm start` 开启本地服务。

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


