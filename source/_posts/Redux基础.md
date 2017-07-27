title: Redux 基础
date: 2017/2/3 11:07:12
categories: iOS
tags:

	- 学习笔记
---

这篇将研究一下 Redux 的基本原理

<!--more-->

## 介绍
### 动机

因为单页应用变得复杂，JavaScript 要管理比任何时候都要多的 state。state 在什么时候，由于什么原因，如何变化变得不受控制。Redux 视图让 state 的变化变得可预测。

### 三大原则

#### 单一数据源

整个应用的 state 被存储在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。这样让同构应用开发变得容易。来自服务端的 state 可以更容易注入到客户端中。

#### State 是只读的

唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。这样确保了视图和网络请求不能直接修改 state。

#### 使用纯函数来执行修改

为描述 action 如何改变 state tree，需要编写 reducers。Reducer 只是一些纯函数，它接收先前的 state 和 action，返回新的 state。你可以把 reducer 拆分为多个小的 reducer，独立地操作 state tree 的不同部分。



## 基础

### Action

Action 是把数据从应用传到 store 的有效载荷。一般你会通过 `store.dispatch()` 将 action 传到 store。

Action 本质上是 JS 普通对象。约定 action内必须使用一个字符串类型的 `type` 字段来表示将要执行的动作。

#### Action 创建函数

在 Redux 中的 action 创建函数只是简单的返回一个 action。这样做将使 action 创建函数更容易被移植和测试。

Action 创建函数也可以是异步非纯函数。

### Reducer

Action 只是描述了有事情发生了这一事实，并没有指明应用如何更新 state。

#### 设计 State 结构

在 Redux 应用中，所有的 state 都被保存在一个单一的对象中。

#### Action 的处理

reducer 就是一个纯函数，接受旧的 state 和 action，返回新的 state。

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

1. 不要直接修改 state。所以这里用 `Object.assign()` 新建了一个副本(`Object.assign()` 会把后面的 set 到第一个参数里，所以这里要放一个空对象)。
2. default 的情况下，返回旧的 state。遇到未知的 action 时，也要返回就的 state。

我们需要修改数组中指定的数据项而又不希望导致突变，因此我们的做法是在创建一个新的数组后，将那些无需修改的项原封不动的移入，接着对需要修改的项用新生成的对象替换。



#### 拆分 Reducer

Reducer 合成是 Redux 应用最基础的模式，可以把一个复杂的 reducer 逻辑拆分到一个单独的函数里。

每个 reducer 只负责管理全局 state 中它负责的那一部分。每个 reducer 的 state 参数都不同，分别对应它管理的那部分 state 数据。

> 其实就是本来有很多 switch…case，判断不同的 action.type 来对 state 的某个部分做不同的操作嘛。每个 state 的部分，可能会在多个 action.type 中都会被修改。
>
> 现在就把处理相同的 state 的部分的操作都拿出来，单独作为一个 reducer。所有会操作这个 state 部分的 action.type 都会调用这个 reducer。然后在这个小的 reducer 中判断到底是那个 action.type 对应怎么处理这部分 state。

Redux 提供了 `combineReducers()` 工具类来完成各个小 reducer 的调用，需要传入各个子 reducer 所组成的对象。

```javascript
import { combineReducers } from 'redux'

const todoApp = combineReducers({
  visibilityFilter,
  todos
})

export default todoApp
```



`combineReducers()` 所做的只是生成一个函数，这个函数来调用你的一些列的 reducer。每个 reducer 会根据它们的 key 来筛选出 state 中的一部分数据并处理。然后这个生成的函数再将所有的 reducer 的结果合并为一个大的对象。

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

#### 展示组件

展示组件值定义外观并不涉及数据从哪里来，如何改变它。传入什么就渲染什么。如果把代码从 redux 迁移到其他架构，这些组件可以直接使用。他们并不依赖于 redux。

#### 容器组件

容器组件将展示组件连接到 redux（相当于一个适配器）。例如，展示型的 `TodoList` 组件需要一个类似 `VisibleTodoList` 的容器组件来监听 Redux store 变化并处理如何过滤出要显示的数据。

技术上讲，容器组件就是使用 `store.subscribe()` 从 Redux state 树中读取部分数据，并通过 props 来吧这些数据提供给要渲染的组件。可以手工开发容器组件，但是建议使用 React Redux 库的 `connect()` 方法来生成，这个方法做了性能优化来避免许多不必要的重复渲染（这样就不用手动实现 React 性能优化建议中的 `shouldComponentUpdate` 方法了）。

因为容器组件是 state 和展示组件的中间件嘛。所以可以在这里按需先处理一下 state，然后再把处理过的 state 传给展示组件。所以可以定义 `mapStateToProps` 函数来指定如何把当前 Redux store state 映射到展示组件中的 props 中。

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

这个在 return 中的 `todos` 就直接成为了该容器组件的 `this.props` 的一员，可以直接通过 `const {todos} = this.props` 获取。

除了读取 state，容器组件还能分发 action。类似的，可以定义 `mapDispatchToProps()` 方法接收 `dispatch()` 方法并返回期望注入到展示组件的 props 中的回调方法。例如，我们希望 `VisibleTodoList` 向 `TodoList` 组件中注入一个叫 `onTodoClick` 的 props 中，还希望 `onTodoClick` 能分发 `TOGGLE_TODO` 这个 action:

```javascript
const mapDispathToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}
```

也就是说，不仅可以把 state 修改好了传进去，也可以把响应时间设置好，再传进去。同样，可以直接通过 `const {onTodoClick} = this.props` 获取这个响应事件。

最后，使用 `connect()` 创建 `VisibleTodoList`,将两个参数传入：

```javascript
import { connect } from 'react-redux'

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

容器组件实例：

![实例](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/容器组件实例.png?raw=true)

#### 传入 Store

所有容器组件都可以访问 Redux Store。可以手动监听 store，但是推荐使用 React Redux 组件 `<Provider>` 来让所有容器组件都可以访问 store，而不必显示的传递它。只需要在渲染根组件时使用即可。

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



### TO BE CONTINUED

暂时先看到这，涵盖了基本的 redux 的原理。后面要用到的继续看。不然太容易忘了。





## 问题

- 为什么要用 redux？
- redux 的三大原则（即特点）是什么？
- 什么是 action？什么是 action创建函数？
- Reducer 是个什么样的函数？基本作用是什么？返回 state 的时候要注意什么？
- `combineReducer` 有什么用？怎么用的？
- `createStore` 需要传入什么？
- redux 的设计核心是什么？有什么好处
- redux 的生命周期是怎样的？
- 什么是展示组件？
- 什么是容器组件？容器组件的作用什么？
- 如何使用 `mapStateToProps` 以及 `mapDispatchToProps`和 `connect` 配合容器组件?
- 为什么要用 `<Provider>`，怎么使用?
- 如何执行异步 action？ 































