title: Redux 基础
date: 2018/6/5 11:07:12
categories: React-Native
tags:

	- 学习笔记
---

记录一下 redux 文档的一些要点。以及学习 redux 中的一些经验。

<!--more-->

## 介绍
### 动机

因为单页应用变得复杂，JavaScript 要管理比任何时候都要多的 state。state 在什么时候，由于什么原因，如何变化变得不受控制。这里产生了矛盾：组件之间需要共享数据，和数据可能被任意修改导致不可预料的结果之间的矛盾。Redux 试图让 state 的变化变得可预测。

### 三大原则

#### 单一数据源

整个应用的 state 被存储在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。

#### State 是只读的

唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。确保视图和网络请求不能直接修改 state。

#### 使用纯函数来执行修改

为描述 action 如何改变 state tree，需要编写 reducers。Reducer 只是一些纯函数，它接收先前的 state 和 action，返回新的 state。你可以把 reducer 拆分为多个小的 reducer，独立地操作 state tree 的不同部分。



## 基础

### Action

Action 是把数据从应用传到 store 的有效载荷。一般你会通过 `store.dispatch()` 将 action 传到 store。

Action 本质上是 JS 普通对象。约定 action内必须使用一个字符串类型的 `type` 字段来表示将要执行的动作。

最简单的 Action 如：

```javascript
const ADD_TODO = 'ADD_TODO'
{
    type: ADD_TODO,
    text: 'build my first redux app'
}
```

#### Action 创建函数

在 Redux 中的 action 创建函数只是简单的返回一个 action。这样做将使 action 创建函数更容易被移植和测试，如：

```javascript
function addTodo (text) {
    return {
        type: ADD_TODO,
        text
    }
}
```

Action 创建函数也可以是异步非纯函数。后面会讨论到。

### Reducer

Action 只是描述了有事情发生了这一事实，并没有指明应用如何更新 state。Reducer 则是用来更新 state

#### Action 的处理

reducer 就是一个纯函数，接受旧的 state 和 action，返回新的 state：

```javascript
(previousState, action) => newState
```

永远不要再 reducer 中执行这些操作：

- 修改传入的参数
- 执行有副作用的操作，如 API 请求或路由跳转
- 调用非纯函数，如 `Date.now()`

reducer 一定要保持纯净。只要传入参数相同，返回计算得到的下一个 state 就一定相同。

```javascript
function todoApp(state = initialState, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return Object.assign({}, state, {
        visibilityFilter: action.filter
      })
      default:
      	return state
  }
}
```

需要注意：

1. 强烈不推荐直接修改 state。所以这里用 `Object.assign()` 新建了一个副本(`Object.assign()` 会把后面的 set 到第一个参数里，所以这里要放一个空对象)。
2. default 的情况下，返回旧的 state。遇到未知的 action 时，也要返回旧的 state。

#### 拆分 Reducer

Reducer 合成是 Redux 应用最基础的模式，可以把一个复杂的 reducer 逻辑拆分到一个个单独的函数里。每个 reducer 只负责管理全局 state 中它负责的那一部分。每个 reducer 的 state 参数都不同，分别对应它管理的那部分 state 数据。

> 其实就是本来有很多 switch…case，判断不同的 action.type 来对 state 的某个部分做不同的操作。每个 state 的部分，可能会在多个 action.type 中都会被修改。
>
> 现在就把处理相同的 state 的部分的操作都拿出来，单独作为一个 reducer。所有会操作这个 state 部分的 action.type 都会调用这个 reducer。然后在这个小的 reducer 中判断到底是那个 action.type 对应怎么处理这部分 state。

Redux 提供了 `combineReducers()` 工具类来完成各个小 reducer 的调用，需要传入各个子 reducer 所组成的对象。

```javascript
import { combineReducers } from 'redux'

// 注意，拆分的 reducer 的 state 不是完整的 state，而是 state 中的属性，和方法名相同，此处就是 visibilityFilter。返回的也不是完整的 state，而只是 visibilityFilter
function visibilityFilter(state = '', action) {
    switch (action.type) {
        case VISIBILITY_FILTER:
            return action.filter
        defaykt:
            return state
    }
}

const todoApp = combineReducers({
  visibilityFilter,		// visibilityFilter 是 state 中的一个属性
  todos					// todos 是 state 中的一个属性
})

export default todoApp

// 调用
todoApp(state = initialState, action) {
    ...
}
```



`combineReducers()` 所做的只是生成一个函数，这个函数来调用你的一系列的 reducer。**每个 reducer 会根据它们的 key（reducer 名） 来筛选出 state 中的一部分数据并处理。然后这个生成的函数再将所有的 reducer 的结果合并为一个大的对象。**(每个小 reducer 拿到的不是整个的 state，而是这个 reducer 对应的 state 中的部分属性。一个 action 的 type 可以影响多个 reducer，最后每个 reducer 生成的 state 中的属性会被合并到 state 中。)

> 没有拆分前是横向的，以 action.type 为基准的。一次 reducer 处理中，能处理多个 state 中的属性。
>
> 拆分之后是纵向的，是以 state 中属性为基准的。一次 reducer 处理只能处理 state 中的一个属性。多次处理后，合成一个完整的 state。

### Store

Store 是把 action 和 reducer 联系到一起的对象。Store 有以下职责：

- 维持应用的 state；
- 提供 `getState()` 方法获取 state；
- 提供 `dispatch(action)` 方法更新 state；
- 提供 `subscribe(listener)` 注册监听器；
- 提供 `subscribe(listener)` 返回的函数注销监听器；

切记，redux 应用只有一个单一的 store。当需要拆分数据处理逻辑的时候，应该使用 reducer 组合而不是创建多个 store。

前面使用 `combineReducers()`  将多个 reducer 合并成一个。现在要将其传递给 `createStore()` 即可完成 Store 的创建。`createStore()` 的第二个参数是可选的，用于设置 state 的初始状态。

### 数据流

严格的**单项数据流**是 redux 架构的设计核心。这意味着应用中所有的数据都遵循相同的生命周期，可以让应用变得可以预测和容易理解。

redux 的生命周期遵循下面四步：

1. 调用 `store.dispatch(action)`
2. Redux store 调用传入的 reducer 函数。将当前的 state 树和 action 传入 reducer。
3. 根 reducer 应该把多个子 reducer 输出合并成一个单一的 state 树。
4. Redux store 保存了根 reducer 返回的完整 state 树。

### 搭配 react

通过 react-redux 将 react 和 redux 融合。组件将被分为展示组件和容器组件。展示组件仅仅是用来展示，它的所有数据都是从 props 中获得。而容器组件则是展示组件的外层封装，用来和 redux 打交道，把 state 传入展示组件。

#### 展示组件

展示组件值定义外观并不涉及数据从哪里来，如何改变它。传入什么就渲染什么。如果把代码从 redux 迁移到其他架构，这些组件可以直接使用。他们并不依赖于 redux。

#### 容器组件

容器组件将展示组件连接到 redux（相当于一个适配器）。例如，展示型的 `TodoList` 组件需要一个类似 `VisibleTodoList` 的容器组件来监听 Redux store 变化并处理如何过滤出要显示的数据。

技术上讲，容器组件就是使用 `store.subscribe()` 从 Redux state 树中读取部分数据，并通过 props 来把这些数据提供给要渲染的组件。可以手工开发容器组件，但是建议使用 React Redux 库的 `connect()` 方法来生成，这个方法做了性能优化来避免许多不必要的重复渲染（这样就不用手动实现 React 性能优化建议中的 `shouldComponentUpdate` 方法了）。

因为容器组件是 state 和展示组件的中间件，所以可以在这里按需先处理一下 state，然后再把处理过的 state 传给展示组件。所以可以定义 `mapStateToProps` 函数来指定如何把当前 Redux store state 映射到展示组件中的 props 中。

例如，`VisibleTodiList` 需要计算传到 `Todolist` 中的 `todos` ，所以定义了根据 `state.visibilityFilter` 来过滤 `state.todos` 的方法，并在 `mapStateToProps` 中使用：

```javascript
const getVisibleTodos = (todos, filter) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed)
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed)  
  }
}

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos,state.visibilityFilter)
  }
}


```

这个在 return 中的 `todos` 就直接成为了该显示组件的 `this.props` 的一员，可以直接通过 `const {todos} = this.props` 获取。

除了显示的数据作为 props 传入，显示组件还需要外部容器组件传入一些事件处理方法。类似的，可以定义 `mapDispatchToProps()` 方法接收 `dispatch` 方法并返回希望被注入到展示组件的 props 中的回调方法。例如，我们希望 `VisibleTodoList` 向 `TodoList` 组件中注入一个叫 `onTodoClick` 的方法到 props 中，这个 `onTodoClick` 能 dispatch  一个 action:

```javascript
const mapDispathToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}
```

也就是说，不仅可以把 state 修改好了传进去，也可以把响应事件设置好，再传进去。同样，可以直接通过 `const {onTodoClick} = this.props` 获取这个响应事件。

最后，使用 `connect()` 创建 `VisibleTodoList`,将两个参数传入：

```javascript
import { connect } from 'react-redux'

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

然后外部就可以直接使用 `VisibileTodoList` 组件了。

容器组件实例：

![实例](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/容器组件实例.png?raw=true)

#### 传入 Store

所有容器组件都可以访问 Redux Store，可以手动监听 store（虽然直接访问会多很多重复代码），但是推荐使用 React Redux 组件 `<Provider>` 来让所有容器组件都可以访问 store，而不必显示的传递它。只需要在渲染根组件时使用即可。

```javascript
import React from 'react'
import { render } from 'react-dom'
import { Provider } from 'react-redux'
import { createStore } from 'redux'
import todoApp from './reducers'
import App from './components/App'

let store = createStore(todoApp)

render(
	<Provider store={store}>
		<APP />
	</Provider>
)
```

#### 关于 react-redux

react 中的每个属性都要通过 props 一层层传递。如果层级很多，会比较麻烦。所以 react 引入了 `context`。某个组件只要往自己的 context 里面放了某些状态，这个组件之下的所有子组件都直接访问这个状态而不需要通过中间组件的传递。

也就是说父组件中设置了 `context`，子组件就可以通过 `this.context.xxx ` 拿到相应属性使用并修改。这就导致了第一个问题，使用 `context` 是非常危险的，子组件可以随意修改。还有另一个问题是，如果每个子组件都是用 `context`，那么组件复用将变得非常困难。

针对问题一， `context` 和 `store` 结合的就非常的完美。`context` 会担心其中数据被随意修改。而 `store` 中的数据只能通过 `dispatch` 修改。
所以可以把 `store` 方到 `context` 中去。

针对问题二，便有了上面的 `connect` 方法，通过显示组件生成容器组件的过程。显示组件只负责根据传入的 `props` 显示，和纯函数一样，便于复用。`connect` 方法生成的容器组件要做的就是获取 `this.context` 中的 `store` 进而 `store.getState()` ,把 map 过的 state 传给显示组件。

这样展示组件中获取 `context` 都的操作都消失了。但是在最外层组件里还有设置 `context` 的操作。为了将这部分脏代码消除。就 react-redux 提供了一个 `Provider` 组件。你只要传入一个 `store`，它就会把它放到 `context` 中去。

## 高级

### 异步 Action

如何把之前定义的同步 action 创建函数和网络请求结合起来？标准做法是使用 `Redux Thunk middleware`。要引入 `redux-thunk` 这个专门的库才能使用。

通过使用指定的 middleware，action 创建函数除了返回 action 对象外，还可以返回函数。这时，这个 action 创建函数就成为了 thunk。

当 action 创建函数返回函数时，这个函数会被 Redux Thunk middleware 执行。

> 其实就是本来要 dispatch 一个 js 对象的 action 的，现在 dispatch 一个函数，这个函数用来获取网络数据，然后在这个函数里 dispatch 请求得到的数据。最开始调用的 dispatch 在判断对象是函数后，就没有其他作用了。
>
> 中间件就是在传入 action 后和在真正 dispatch 前插入一些逻辑。thunk 中间件就是插入了判断 dispatch 传入的是不是函数的逻辑，是函数就调用，不是函数，就和普通方式一样直接 dispatch

我们要如何在 dispatch 机制中引入 Redux Thunk middleware 呢？使用 `applyMiddleware`:

```javascript
impoort thunkMiddleware from 'redux-thunk'
const store = createStore(
	rootReducer,
	applyMiddleware(
    	thunkMiddleware
    )
)

store.dispatch(fetchPosts('reactjs')).then(() => console.log(store.getState()))	// fetchPosts 函数是一个自定义的函数，在里面会进行网络请求
```

> 并不是说 thunk 是唯一解决方式，还可以使用 redux-promise，redux-promise-middleware。
>
> 需要了解的是这种处理异步的方式。

### Middleware

我们知道，我们 `store.dispatch(action)` 后，` dispatch` 方法中会调用 `reducer` 。但是我们很多时候希望在执行 `reducer` 之前或者之后做一些其他的操作，比如记录日志等，这个时候就需要使用 middleware。

middleware 所做的就是替换 redux 提供的 `dispatch` 方法:

```javascript
let next = store.dispatch
store.dispatch = function dispatchAndLog(action) {
    console.log(action)
    let result = next(action)
    console.log(store.getState())
    return result
}
```

可以看到，现在 `store.dispatch` 一个 action 的前后都会执行 log。并且现在的 `dispatch` 还会返回原本的 dispatch 的返回值(虽然这里原本的 dispatch 是 redux 的 dispatch ，不反回任何，但是如果 middleware 修改过的 dispatch 可能会返回 result，所以还是需要 return 的)

现在我们考虑有多个 middleware 需要应用。我们需要把上一个 middleware 修改过的 `dispatch` 传入 下一个 middleware。那么考虑这个中间件方法需要几个参数：`store`，上一个中间件修改过的 `dispatch`，`action`。所以我们可以采用函数式：

```javascript
const logger = store => next => action => {
    console.log(action)
    let result = next(action)
    console.log(store.getState())
    return result
}

const thunk = store => next => action => {
    typeof action === 'function'
    	? action(store.dispatch, store.getState)
    	: next(action)
}
```

这样，在 `createStore()` 的时候，不只是传入一个 `reducer`，还包括一个中间件参数：

```javascript
let store = createStore(
	todoApp,
    applyMiddleware(
    	logger,
        thunk,
    )
)
```

`applyMiddleware` 应该做的是这样一个操作：

```javascript
let dispatch = store.dispatch
for temp in middlewares {
    dispatch = temp(store)(dispatch)
}
```

### 组织 reducer

对于单个的 reducer，项目复杂后，reducer 可能会变得非常的长：

```javascript
function appReducer(state = {}, action) {
    switch(action.type) {
        case 'ADD_TODO' : {
            return Object.assign({}, state, {
                todos: state.todos + action.add
            })
        }
        case 'TOGGLE_TODO' : {
            return Object.assgin({}, state, {
                todos: state.todos + action.toggle
            })
        }
        case 'SET_VISIBILITY_FILTER' : {
            return Object.assign({}, state, {
                visibilityFilter: action.filter
            })
        }
        ...
    }
}
```

可以看到，这样的代码含有两个问题：

- 多个字段的处理逻辑都放在一个方法里，不够清晰
- 会包含非常多的 switch...case... 方法，很多冗余代码

#### 分离字段

针对第一个问题我们可以为每个字段都实现一个子 reducer，然后把各部分返回的结果组合：

```javascript
function setVisibilityFilter(visibilityState, action) {
    return action.filter
}

function addTodo(todoState, action) {
    return todoState + action.add
}

function toggleTodo(todoState, action) {
    return todoState + action.toggle
}

function visibilityReducer(visibilityState = 'SHOW_ALL', action) {
    switch(action.type) {
        case 'SET_VISIBILITY_FILTER' : return setVisibilityFilter(visibilityState, action)
        default : return visibilityState
    }
}

function todosReducer(todoState, action) {
    switch(action.type) {
        case 'ADD_TODO' : return addTodo(todoState, action)
        case 'TOGGLE_TODO': return toggleTodo(todoState, action)
		default : return todosState
    }
}

function appReducer (state = {}, action) {
    return {
        todos: todosReducer(state.todos, action)
        visibilityFilter: visibilityReducer(state.visibilityFilter, action)
    }
}
```

这样的写法把各个字段分别处理，逻辑就清晰了很多。

#### 减少样板代码

针对问题二提到的 `switch...case…` 产生的样板代码，js 的语言特性可以很好地消除：

```javascript
// 消除样板代码的主要方法
function createReducer(initialState, handlers) {
    return function reducer(state = initialState, action) {
        if (handlers.hasOwnProperty(action.type)) {
            return handlers[action.type](state, action)
        } else {
            return state
        }
    }
}

const todoReducer = createReducer({}, {
    'ADD_TODO' : addTodo,
    'TOGGLE_TODO' : toggleTodo,
})

function appReducer (state = {}, action) {
    return {
        todos: todosReducer(state.todos, action)
        visibilityFilter: visibilityReducer(state.visibilityFilter, action)
    }
}
```

用一个对象的键来替代 case，判断 `hasOwnProperty` 来替代是否命中 case。

这里就已经非常像 redux 提供的  `combineReducer`  方法的实现了

## 常见问题

### Redux 只能搭配 React？

Redux 可以作为任何 UI 层的 store。但是一般在结合 UI 随 state 变化时，Redux 能发挥最大的作用。

### Reducer

#### 如何在 reducer 间共享 state？`combineReducers` 是必须的吗？

redux store 推荐结构是将 state 分成"层"，并提供独立的 reducer 方法管理各自的数据层。

许多用户想要在 reducer 之间共享数据，但是 `combineReducers`不允许此种行为。如果一定要这样做，你需要：

- 重构 state 层，让单独的 reducer 处理更多的数据。
- 自定义方法处理这些 action，用自定义的顶层 reducer 方法替换 `combineReducers` 。使用 `combineReducers` 处理尽可能多的 action，同时为存在 state 交叉部分的 action 执行更专用的 reducer。
- 类似于 redux-thunk 的异步 action 创建函数可以通过 `getState()` 方法获取所有的 state。action 创建函数能从 state 中检索到额外的数据并传入 action。所以 reducer 有足够的信息去更新 state 层。

#### 处理 action 必须使用 switch 语句嘛？

不必要，但是 switch 是最常用的方式。也可以用 if 或者上面组织 reducer 中使用的方式。

### State

#### 是否所有 state 都维护在 redux 中？

一般来说业务数据放在 redux 中，页面数据则不必要。

### Store

#### 可以创建多个 store 嘛？

最好不要创建多个 store，而是尝试组合 reducer。

#### 能在组建里直接使用 store 并使用吗？

最好不要直接获取 store 实例。借助 react-redux，由 `connect()` 将 `<Provider store={store}>` 传入的 store 通过 `mapStateToProps` 传入子组件。 

### 性能

#### 每个 action 都调用所有的 reducer 会不会很慢？

不会

#### reducer 中必须对 state 进行深拷贝么？

你需要创建一个 state 的副本，但是 state 中的没有变化的数据，还是使用浅拷贝，变化的才会创建新的引用。

### react redux

#### 为何组件没有被重新渲染，或者 `mapStateToProps` 没有运行？

action 分发后没有重新渲染，最主要的原因是对 state 进行了直接修改。React Redux 会在 `shouldComponentUpdate` 中对新的 props 进行浅层的判等检查。如果所有的引用都是相同的，那么返回 false 从而跳过此次组价你的更新。

所以，要注意，不管何时更新了一个嵌套值，都必须返回上层的数据的副本给 state 树。即如果数据是 `state.a.b.c.d` 中的 `d` 变化，你也必须返回 `c`,`b`,`a`,`state` 的拷贝。 

#### 为何组件频繁重新渲染？

react redux 做了优化，会对 `mapStateToProps` 传入的 props 判等。但是如果每次 `mapStateToProps` 都生成新的对象实例的话，这种判等就没有用处了。比如：

```javascript
const mapStateToProps = (state) => {
    return {
        objects: state.objectIds.map(id => state.objects[id])
    }
}
```

通过 `map` 会生成新的实例。所以判等也就不可能成功了。

这种可以使用 Reselect，在 `mapStateToProps` 前就进行判断是否需要 map state。

#### redux 应该只连接到顶层组件嘛？

并不是。当你感觉到你是在父组件里通过复制代码为某些子组件提供数据时，就可以抽出一个容器组件了。只要你认为父组件过多了解子组件的数据或者 action，就可以抽取容器。



## Hanzo 

Hanzo.js 是一个借鉴了 Dva.js  react 框架。集成了路由以及 redux。这里主要讨论 redux 相关部分的 RN 中的实现。

首先在 RN 的入口 index.ios.js 中：

```javascript
import Hanzo from 'hanzojs/mobile'
import React, { Component } from 'react'

const App = new Hanzo()
App.registerModule(require('./modules/properties'))
App.registerModule(require('./modules/todoApp'))

const AppComponent = App.start()

class CRM extends Component {
  constructor(props) {
    super(props);
  }

  render() {
    return <AppCompinent />
  }
}
    
React.AppRegistry.registerComponent('reactNativeCrm', () => CRM);
```

通过 hanzo 提供的初始化方法，可以创建一个包含所有信息的实例，相当于是之后所有视图以及逻辑的根：

```javascript
const app = {
  // properties
  _models: [],
  _reducers: {},
  _views: {},
  _router: null,
  _routerProps: {},
  _routes: null,
  _store: null,
  _history: null,
  _isomorphic: hooks.isomorphic || false, // server-render options

  // methods
  use, // add redux-middlewares, extra-reducers, etc
  registerModule, // register a module
  router, // config router
  start, // start hanzo instance
  getStore, // get redux store
  getRoutes, // get router config
  getModule, // get the register module
};
```

通过 `registerModule` 将各个













