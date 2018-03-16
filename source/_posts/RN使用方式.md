title: React-Native 使用方式
date: 2016/11/25 10:07:12  
categories: React-Native
tags:
	- React-Native
---

这里基于0.39的官方文档，列举了一些需要知道的点。

<!--more-->
### AppRegistry
`index.ios.js` 作为一个组件的注册集合以及入口，在这里，将所有组件像这样注册。

```javascript
AppRegistry.registerComponent('HelloApp', () => HellodApp);
AppRegistry.registerComponent('HelloWorldApp', () => HelloWorldApp);
```

### Props


#### 自定义组件的 Props
自定义组件可以提高复用性。使用时在函数中引用 `this.props.属性名`，例如下面的 `name` 属性：

```javascript
class Greeting extends Component {
  render() {
    return (
      <Text>Hello {this.props.name}!</Text>
    );
  }
}

class LotsOfGreetings extends Component {
  render() {
    return (
      <View style={{alignItems: 'center'}}>
        <Greeting name='Rexxar' />
        <Greeting name='Jaina' />
      </View>
    );
  }
}
```

### State
通过 `Props`,`State` 控制组件。`Props` 在父组件中指定，一经指定，不能改变。对于需要改变的组件，要使用 `state`。

一般要在 `constructor` 中初始化 `state`(通过 `getInitialState` 方法为es5方法，已被淘汰)，需要修改时调用 `setState`。

```javascript
class Blink extends Component {
  constructor(props) {
    super(props);
    this.state = { showText: true };

    // 每1000毫秒对showText状态做一次取反操作
    setInterval(() => {
      this.setState({ showText: !this.state.showText });
    }, 1000);
  }

  render() {
    // 根据当前showText的值决定是否显示text内容
    let display = this.state.showText ? this.props.text : ' ';
    return (
      <Text>{display}</Text>
    );
  }
}

class BlinkApp extends Component {
  render() {
    return (
      <View>
        <Blink text='I love to blink' />
        <Blink text='Yes blinking is so great' />
      </View>
    );
  }
}
```

这个例子中 text 是固定的，所以用 `Props` 在父控件创建的时候传入。在构造函数的时候创建了 `State` 和其属性 `showText`。当然，还可以再这里初始化任意多个属性，用逗号隔开，相当于创建一个 `state` 对象。使用的时候直接可以点出来：`this.state.showText`。想要修改的时候调用 `this.setState()` 方法，将要修改的属性组成对象传入。

拓展：可以学习一下状态容器如：`Redux`。另外，关于 `state` 的更多原理，可以参考 `React.js`。

### StyleSheet
相当于一个样式合集：

```javascript
import { AppRegistry, StyleSheet, Text, View } from 'react-native';

const styles = StyleSheet.create({
  bigblue: {
    color: 'blue',
    fontWeight: 'bold',
    fontSize: 30,
  },
});

//使用：
<View style={styles.bigblue}/>
```

### flex
在 `style` 中通过flex可以灵活地将组件撑满整个屏幕。

组件能够撑满剩余空间的前提是其父容器的尺寸不为零。如果父容器既没有固定的 `width` 和 `height`，也没有设定 `flex`，则父容器的尺寸为零。其子组件如果使用了 `flex`，也是无法显示的。

### flexbox布局
#### flexDirection
在 `style` 中指定布局的**主轴**，是 `row` 还是 `column`。默认是 `column`。

#### justifyContent
在 `style` 中指定元素沿着**主轴**的排序方式。**注意**，这个属性是定义在父控件，作用在子控件上的。

包括 `flex-start`,`flex-end`,`center`,`space-between`,`space-around`。

前面三个不说了，后面两个说一下：
1. `space-between`:左右各一个子控件，剩余的子控件均匀的分布在中间。如果只有一个，那么这个居左。
2. `space-around`:所有子控件均匀的分布，如果只有一个，那么这个居中。

#### alignItems
在 `style` 中指定布局的**次轴**的排序方式。**注意**，这个属性是定义在父控件，作用在子控件上的。

包括 `flex-start`,`flex-end`,`center`,`stretch`。

其中 `stretch` 表示拉伸撑满次轴，前提是不设置子控件宽度。

### 布局样式
#### 属性
除了上面说的那些，再摘取一些有用的 `style` 中的属性：

1. `border` 系列:设置边框的宽度，`borderTopWidth`,`borderLeftWidth`...上下左右都可以分别设置。
2. `alignSelf`:类似于 `alignItems`，只不过这个属性定义在子控件中，重写父控件的 `alignItems`。因为子控件在次轴上没有必要完全按照父控件的控制，那样太死板了，`alignSelf` 的目的就是搞这个特殊。
3. `left` 系列:定义子控件到父控件的左边距。**注意**，要先设置 `position:absolute` 否则无效。类似的还有 `right`,`top`,`bottom`。
4. `flexWrap`:当子控件到底后自动换行。`wrap` 为自动换行，`noWrap` 为不换行。**注意**，要定义在父控件中。
5. `margin` 系列:设置子控件到相邻子控件的间隔。
6. `max`,`min` 系列:设置控件的最大(小)宽度(高度).
7. `padding` 系列:在父控件内设置，设置子控件的起点。
8. `position`:分为 `absolute` 绝对，`relative`相对。默认是相对的，如果要使用 `left` 等样式，就要使用绝对布局。**绝对布局的子控件的大小不会影响父控件的大小**
9. `zIndex`: 控制控件的绘制层级，该属性越大，越绘在顶层。

#### 使用
一般使用方式:
```jsx
style={{height:xx,width:xx}}
```

如果有多个对象，需要用数组的方式传入：

```jsx
style={[{height:xx,width:xx,},{height:yy,wudth:yy,}]}
```

如果有相同的属性重复赋值，则以后一个覆盖前一个为准。

### TextInput
#### 例子
文本输入组件，先看一个例子：

```javascript
class UselessTextInput extends Component {
  render() {
    return (
      <TextInput
        {...this.props} // Inherit any props passed to it; e.g., multiline, numberOfLines below
        editable = {true}
        maxLength = {40}
      />
    );
  }
}

class UselessTextInputMultiline extends Component {
  constructor(props) {
    super(props);
    this.state = {
      text: 'Useless Multiline Placeholder',
    };
  }

  render() {
    return (
     <View style={{
       backgroundColor: this.state.text,
       borderBottomColor: '#000000',
       borderBottomWidth: 1 }}
     >
       <UselessTextInput
         multiline = {true}
         numberOfLines = {4}
         onChangeText={(text) => this.setState({text})}
         value={this.state.text}
       />
     </View>
    );
  }
}
```

这里为什么要把这个例子贴出来呢？主要有**两个要点**：

首先，`{...this.props}` 的语法。这里`{}` 还是表示使用JS语法的意思。`...` 是es6的语法，叫做**展开运算符**，该功能用来遍历对象。可以看看下面这个例子：

```javascript
var array = [1,2,3,4,5,6,7];
console.log(array);
//输出 [1, 2, 3, 4, 5, 6, 7]
console.log(...array);
//输出 1 2 3 4 5 6 7
```

`this.props` 是个对象，肯定不能放在 Component 中，但是用了**展开运算符**，里面的属性就全部被展开了，也就符合了 JSX 的语法。不只是 `this.props` **任何对象都可以用这种展开的方式将参数传入 Component**。不过不建议滥用，因为容易将很多多余参数传入，可能产生一些不易排查的问题。


其次，`onChangeText={(text) => this.setState({text})}`。一般 `setState()` 中都是对象，为什么这里只是 `text`？这也是es6的语法，在键和值相等的时候可以省略 `{text:text}` 为 `{text}`。也就是本例的 `this.setState({text})`。

和 `onChangeText` 相对应的还有一个 `onEndEditing` 属性。前者是每次 text 变化的时候都会回调；后者只有在结束输入的时候会回调一次。

#### 部分属性
1. `autoFocus`:在 `componentDidMount` 后是否自动获得焦点。
2. `defaultValue`:提供文本框的初始值。
3. `editable`:文本框是否可编辑
4. `maxLength`:文本框中最长字符数。使用这个属性而不用JS逻辑去实现，可以避免闪烁的现象。
5. `multiline`:是否可以多行，默认 false
6. `onFocus`,`onBlur`,`onSubmitEditing`:输入框在各种情况下的回调函数。
7. `onChange`:当输入框内容改变时，回调该函数
8. `onChangeText`,`onEndEditing`:前者当输入框内容改变时，回调该函数。改变后的文字内容将作为参数传入;后者当输入框结束编辑后回调该函数。
9. `placeholder`:占位符
10. `placeholderTextColor`:占位符颜色.(注意，貌似没有 placeholderTextSize 这个属性)
11. `value`:输入框内显示的值 
12. `underlineColorAndroid`:这是 android 特有的属性，非常的坑。文档里是这么说的“文本框的下划线颜色”，其实不是这个意思，这个属性设置的是整个文本框的颜色。比如在 iOS 上，直接设置 `backgroundColor='red'` 就能把文本框设成红色，但是 android 不然，在 background 之上还有一层，就是这个 `underlineColorAndroid` 需要将这个也设置成红色，才能正确显示成红色。
13. 这里还有一个坑点，就是一定一定一定设置 TextInput 的高度。不然即使 TextInput 大小为0，任然会显示 placeholder ，这就会导致一直没有点击效果。所以**一个调试技巧：给控件设置醒目的背景颜色，这样有助于了解控件的布局情况，方便调试。**

#### 方法
1. `isFocusd()`:判断当前输入框是否获得焦点。
2. `clear()`:清空输入框内容。


### ScrollView
#### 属性
1. `contentContainerStyle`:比较重要的属性，相当于在 ScrollView 中内嵌了一个 View，这个属性就是控制这个内嵌 View 的。
2. `horizontal`:内容水平排布，不知道和 `flexDirection` 有什么区别
3. `keyboardDismissMode`:用户拖拽视图是否隐藏软键盘。`none`,`on-drag`
4. `keyboardShouldPersistTaps`:控制是否在当点击文本输入框以外区域时，自动隐藏软键盘。true 不隐藏，false 隐藏。默认 false
5. `onScroll`:滚动的时候以一定频率回调该函数。
6. `refreshControll`:指定 RefreshController 组件，用于为 ScrollView 提供下拉刷新功能。
7. `pagingEnabled`:滚动条停在整数倍位置。
8. `scrollEventEnabled`:控制视图能否滚动。

#### 方法
1. `scrollTo`:滚动到指定的x, y偏移处。第三个参数为是否启用平滑滚动动画。

```javascript
scrollTo({x: 0, y: 0, animated: true})
```

#### RefreshControl
这一组件可以用在 ScrollView 或 ListView 内部，为其添加下拉刷新的功能。当 ScrollView 处于竖直方向上的起点位置时，此时下拉会触发一个 `onRefresh` 事件。有两个常用属性：

1. `onRefresh`:在视图开始刷新时调用。
2. `refreshing`: 视图是否应该在刷新时显示指示器。一般是从网络获取数据的时候是 true，获取完数据为 false

#### 示例：用Scrollview实现带下拉刷新的Tableview

```javascript
// 每一个row的view
const Row = React.createClass({
  _onClick: function() {
    this.props.onClick(this.props.data);
  },
  render: function() {
    return (
     <TouchableWithoutFeedback onPress={this._onClick} >
        <View style={styles.row}>
          <Text style={styles.text}>
            {this.props.data.text + ' (' + this.props.data.clicks + ' clicks)'}
          </Text>
        </View>
      </TouchableWithoutFeedback>
    );
  },
});

const RefreshControlExample = React.createClass({
  statics: {
    title: '<RefreshControl>',
    description: 'Adds pull-to-refresh support to a scrollview.'
  },

  getInitialState() {
    return {
      isRefreshing: false,
      loaded: 0,
      rowData: Array.from(new Array(20)).map(
        (val, i) => ({text: 'Initial row ' + i, clicks: 0})),
    };
  },

  _onClick(row) {
    row.clicks++;
    this.setState({
      rowData: this.state.rowData,
    });
  },

  render() {
    const rows = this.state.rowData.map((row, ii) => {
      return <Row key={ii} data={row} onClick={this._onClick}/>;
    });
    return (
      <ScrollView
        style={styles.scrollview}
        refreshControl={
          <RefreshControl
            refreshing={this.state.isRefreshing}
            onRefresh={this._onRefresh}
            tintColor="#ff0000"
            title="Loading..."
            titleColor="#00ff00"
            colors={['#ff0000', '#00ff00', '#0000ff']}
            progressBackgroundColor="#ffff00"
          />
        }>
        {rows}
      </ScrollView>
    );
  },

  _onRefresh() {
    this.setState({isRefreshing: true});
    setTimeout(() => {
      // prepend 10 items
      const rowData = Array.from(new Array(10))
      .map((val, i) => ({
        text: 'Loaded row ' + (+this.state.loaded + i),
        clicks: 0,
      }))
      .concat(this.state.rowData);

      this.setState({
        loaded: this.state.loaded + 10,
        isRefreshing: false,
        rowData: rowData,
      });
    }, 5000);
  },
});
```

省略了styleSheet。这里主要注意几点：
1. 譬如封装一个组件 Row，其中有点击事件的处理。iOS 中一般使用代理。这里由于js的特性，可以把外部的点击方法传入组件内，再由组件的点击事件调用。
2. 注意 `map()` 方法的使用。这也就是用 scrollview 实现 tableview 的方式。
3. 在 `render` 里可以插入js方法，如上面的 `{row}`,只要方法返回的是JSX的 component就行。（貌似component里的属性只能用展开运算符才能插入，用js方法返回的方式一直都报错）
4. `RefreshControl` 看起来也就是一个 component，由 scrollview 控制什么时候展示，怎么展示。
5. 如果想要自定义上拉刷新，就得知道 `contentOffset` 可以研究一下。

### ListView
ListView设计到各种优化，以及各种方法，掌握的不是很好，需要进一步学习。
#### 属性
1. ScrollView 全部属性
2. `dataSource`:`ListView.DataSource实例`。
3. `initialListSize`:指定在组件刚挂载的时候渲染多少行数据。(试了下没看出有什么效果。)
4. `onChangeVisibleRows`:显示区域的 cell 变化时调用。（api 和实际使用有出入，需要实际使用的时候检验，不过这个东西好像没什么太大用）
5. `onEndReachedThreshold`:调用 `onEndReached` 之前的临界值.
6. `onEndReached`:滚动到底部时的回调，如果数据不满足一屏，会立即回调，需要手动标记过滤。
7. `pageSize`:每次事件循环（每帧）渲染的行数。默认是1。（好像不需要自己设置，不知道设了有什么影响）
8. `removeClippedSubviews`:说是提升性能默认开启，不知道具体有啥用。
9. `renderHeader`,`renderFooter`:页头，页脚。
10. `renderRow`: cell 的实现，方法为`(rowData, sectionID, rowID, highlightRow) => renderable`
11. `renderSeparator`: `(sectionID, rowID, adjacentRowHighlighted) => renderable` 一个可渲染的组件会被渲染在每一行下面.如果被点击了，就会显示出来。没什么太大用。如果要用，要在 return 的 view 里添加 `key`，最好看一下示例。
12. `scrollRenderAheadDistance`:距离多少开始渲染下一行。
13. `renderScrollComponent`:不知道怎么用。
14. `renderSectionHeader`:每个小节(section)渲染一个粘性的标题。`(sectionData, sectionID) => renderable` 其中 `sectionData` 就是section内内容的数组。

#### 一些方法
1. listview 只是提供了渲染其中 cell 的功能，默认是 cell 竖着排下来。那么如果要实现成 gridview 之类该怎么做呢？就是给 listview 附上从 scrollview 中继承来的 `contentContainerStyle` 这个 style 用来控制其中子视图的样式，只要在其中将 `flexDirection:‘row’,justifyContent: 'space-around',flexWrap: 'wrap'` 这样一设置。就能达到横排 cell，并且自动换行的效果。

### 网络
感觉没什么好写的，暂且不表

### navigator
暂时还是用原生的 navigator，感觉 navigator 不太好用，所以暂且不表。

如果需要使用，可以查看这篇[教程](http://bbs.reactnative.cn/topic/20/新手理解navigator的教程)










