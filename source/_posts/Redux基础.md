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

// 注意，拆分出来的 reducer 中的 state 不是完整的 state，而是 state 中的属性，和方法名相同，此处就是 visibilityFilter。返回的也不是完整的 state，而只是 visibilityFilter
function visibilityFilter(state = '', action) {
    switch (action.type) {
        case VISIBILITY_FILTER:
            return action.filter
        default:
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

这样的写法把各个字段分别处理，逻辑就清晰了很多。`combineReducers` 其实做的就是同样的事。

#### 减少样板代码

针对问题二提到的 `switch...case…` 产生的样板代码，js 的语言特性可以很好地消除：

```javascript
function addTodo(todoState, action) {
    return todoState + action.add
}

function toggleTodo(todoState, action) {
    return todoState + action.toggle
}

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
    addTodo,
    toggleTodo,
})

function appReducer (state = {}, action) {
    return {
        todos: todosReducer(state.todos, action)
        visibilityFilter: visibilityReducer(state.visibilityFilter, action)
    }
}
```

用方法名来替代 case，也就是替代 `action.type`。然后把这些方法合在一起生成一个 reducer。这种方式就是类似 redux-actions 中 `handleActions` 的实现方式。

> 这一小节其实是非常重要的。介绍了两个 reducer 的处理方式
>
> 1. 将一个 reducer 拆分成多个小 reducer，每一个 reducer 只负责 state 中相对应的一部分。这样的拆分可以是多层级的，即小 reducer 可以拆分成很多很多小小 reducer。具体见下图
> 2. 在每一个最小粒度的 reducer 中，又根据不同的 `action.type` 拆分出许多与之对应的方法。这些方法组合成 reducer。redux 会遍历所有的 reducer 找出对应 `action.type` 的处理方法，所以不同模块一般要在这些方法前加上 namespace 的前缀，否则容易一个 action 触发多个同名(`action.type` 相同)方法。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_1.png?raw=true)

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

Hanzo.js 是一个借鉴了 Dva.js  的框架。集成了 router 以及 redux。这里主要讨论 redux 相关部分的 RN 中的实现。

### 需求分析

在分析一个框架的时候，我们需要思考，这个框架需要帮助使用者完成那些步骤，以及需要使用者提供哪些东西。

通过前面 redux 的学习，我们可以发现，用户没有必要知道 Store 是如何创建的（`createStore`），没有必要知道 reducer 是如何合成的(`combineReducer`)，没有必要知道 redux 分发事件  `dispatch`,没有必要知道根视图 `Provider`，没有必要知道 `state` 树如何形成的，没有必要知道 `reducer` 和  `state` 中属性一一对应，没有必要知道 `mapStateToProps` 以及 `mapDispatchToProps`，没必要知道 `applyMiddleware`，没必要知道 `action.type` 判断的 switch...case...。 

需要使用者提供的是，哪些方法以及哪些属性是需要作为 props 传入模块的， props 中的方法需要执行什么异步操作(`action`)，以及如何影响 props 中的属性的(`state`)。

带着上面的想法，我们可以大概推测一下 hanzo 做了什么。

1. 为每一个模块设置一个 reducer。这个 reducer 应该和 state 中相应属性同名。每个 reducer 中应该包含很多的方法，这些方法代表着不同的 `action.type`  的处理方法。为了和其他模块的 `action.type` 的处理方法不重名，还需要为这些方法添加模块特有的前缀。在 `dispatch(action)` 的时候，`type` 就要是包含前缀的方法名。
2. 隐藏任何关于 redux 的东西。state 不需要从 store 中取。设置 state 的时候，也不要出现 `dispatch` 方法。

### 使用

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
    return <AppComponent />
  }
}
    
React.AppRegistry.registerComponent('reactNativeCrm', () => CRM);
```

通过 Hanzo 实例的 `registerModule` 方法可以注册模块。模块示例如下：

```javascript
import { connect } from 'hanzojs'
import model from './model'

module.exports = {
  models: model,
  views: {
    TodoApp: connect((state) => {
      return {
        ...state.todoApp,
        todos: state.todoApp.todos.filter(todo => {
          if (state.todoApp.filter === VisibilityFilters.ALL) {
            return true;
          } else if (state.todoApp.filter === VisibilityFilters.COMPLETED) {
            return todo.completed;
          } else if (state.todoApp.filter === VisibilityFilters.INCOMPLETE) {
            return !todo.completed;
          }
        })
      }
    }, model)(require('./views/index.js'))
  }
}
```

引入了和方法相关的 model 以及和视图相关的 view。导出 models 主要是为了导出模块的 `state` 以及 `reducer`，以此创建 `store`。导出 view 的时候，使用了 hanzo 提供的 connect 把两者组合，其实就是相当于设置里 `mapStateToProps` 和 `mapDispatchToProps`。

一个标准的 model 写法如下：

```javascript
module.exports = {
  namespace: '',
  state: {
  },
  handlers: [
  ],
  publicHandlers: [
  ],
  reducers: {
  },
}
```

其中，namespace 就是当前模块名，对于多层级的 namespace，hanzo 会将其转为多层级的对象结构。namespace 会被作为当前模块的在全局的 `state` 树的名字，以及相对应的 `reducer` 的名字。

state 为当前模块的初始状态，也就是在 `state` 树中属性的具体值。

handlers 为当前模块能调用的方法，会以 `mapDispatchToProps` 的形式传入 view。publicHandlers 为全局都能调用的方法，所以会加入到每一个模块的 `mapDispatchToProps` 中去。

reducers 会被 hanzo 做进一步处理。先将数组中的各个方法组合成该模块的 reducer，然后将这些 reducer `combineReducer` 为一个大的 reducer。**这里 reducers 里的各个方法分别对应 `action` 的不同 type。**

注册完模块，通过 hanzo 实例的 start 方法，返回一个视图，作为 RN 的根视图。

另外还有一块关于 middlerware 的方面。使用 hanzo 实例的 `use` 方法注册中间件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_2.png?raw=true)



### 原理

#### 注册相关（主要是设置 reducer）

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

随后我们会调用 `registerModule` 方法注册模块。注册模块的主要逻辑如下：

```javascript
if (isPlainObject(module.models)) {
  let Actions = {}
  let namespace = module.models.namespace.replace(/\/$/g, ''); // events should be have the namespace prefix
  Object.keys(module.models.reducers).map((key) => {
    if (key.startsWith('/')) { // starts with '/' means global events
      // 如果reducer 以 '/' 开头，那么就不会添加命名空间，表示用户明确知道需要哪个 type 触发。
      Actions[key.substr(1)] = module.models.reducers[key];
    } else {
      Actions[namespace + '/' + key] = module.models.reducers[key];
    }
  })
  let _temp = handleActions(Actions, module.models.state)
  let _arr = namespace.split('/')
  _mergeReducers(this._reducers, _arr, _temp)
}

if (module.publicHandlers && Array.isArray(module.publicHandlers)) {
  module.publicHandlers.map((item) => {
    GlobalContext.registerHandler(namespace.replace(/\//g, '.') + '.' + item.name, item.action)
  })
}

/**
 * private method
 * merge reducers by hierachy
 * user/login, user/info -> user:{ login, info }
 */
function _mergeReducers(obj, arr, res) {
  if(arr.length > 1) {
    let hierachy = arr.splice(0,1)[0]
    obj[hierachy] = obj[hierachy] || {}
    _mergeReducers(obj[hierachy], arr, res)
  } else {
    obj[arr[0]] =  res || {}
  }
}
```

前面说过 `reducers` 数组中的方法对应的是不同的 `action.type`。为了区分不同模块的 `action.type`，需要为其中的每个方法都添加上 `namespace`。但是还有一些方法就不需要添加 `namespace`，因为这些方法表示的 `action.type` 是为了响应其他模块发出的 dispatch 而创建的。如下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_3.png?raw=true)

随后将处理好的 `reducers` 通过 redux-actions 的 `handleActions` 方法，将其转化为一个 reducer。相当于把一个个独立的方法转化为 switch...case... 了。

> 创建 reducer 的时候，其实就已经把 `defaultState` 传进去了。所以后面创建 store 的时候就不需要总的 `initialState` 了

之后通过 `_mergeReducers` 方法把上面生成的各个 reducer 放到 hanzo 实例的 `_reducers` 的各个命名空间下。比如 `namespace` 为 `order/newOrder` 的模块，它的 reducer 就放在 hanzo 实例对象的 `_reducers.order.newOrder` 下，等待最后的 `combineReducer`。

最后，把 `publicHandlers` 中的所有方法保存到 `GlobalContext` 中去。`publicHandlers` 中的方法所有模块都能通过 `GlobalContext` 获取并使用：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_5.png?raw=true)

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_4.png?raw=true)

注册模块的逻辑就是这样。现在需要把各个部分糅合到一起。hanzo 提供了 `start` 方法：

```javascript
/**
 * start the whole hanzo instance
 * return React.Component
 */
function start(container) {
  const me = this
  const AppNavigator = me._router; // react-navigation
  let store = getStore.call(me)
  const isomorphic = me._isomorphic
  const App = ({ dispatch, nav }) => (
    <AppNavigator navigation={addNavigationHelpers({ dispatch, state: nav })} />
  );
  const mapStateToProps = state => ({
    nav: state.nav,
  });

  const AppWithNavigationState = connect(mapStateToProps)(App);

  return class extends Component {
    render() {
      isomorphic ? store = getStore.call(me) : null
      return (
        <Provider store={store}>
          <AppWithNavigationState />
        </Provider>
      ) 
    }
  } 
}
```

`start` 整合了 redux-react 的功能。主要创建了 store 以及提供了 `Provider`。

这里有一部分 navigation 相关功能。`App` 是一个显示组件，至于这种写法，其实就相当于router 组件需要 store 的 `dispatch` 方法以及 state 的 `nav` 属性作为 props 传入。然后的 `connect` 方法也确实是这样做的。`connect` 方法创建了一个容器组件 `AppWithNavigationState` ，然后将其作为 `Provider` 的子组件。

再回到创建 store 的过程：

```javascript
/**
 * create the redux-store
 */
function getStore() {
  let middlewares = plugin.get('onAction');

  let enhancer = applyMiddleware(...middlewares)
  if (typeof __DEV__ !== 'undefined' && __DEV__) { // dev mode
    const devTools = plugin.get('dev') || ((noop) => noop)
    if(devTools.apply) {
      enhancer = compose(
        applyMiddleware(...middlewares),
        devTools
      )
    }
  }

  const createAppStore = enhancer(createStore);
  
  this._store = Object.assign(this._store || {}, createAppStore(getReducer.call(this), initialState));
  return this._store
}
```

可以看到，先用 `applyMiddleware` 生成了中间件，中间件可以通过 hanzo 的 `use` 方法注册，一般会注册的中间件有： 

- redux-thunk：用来处理异步 action
- redux-promise-middleware：为异步请求的 `action.type` 加上后缀。例如：`Loading`,`Success`,`Error` 等。

这里 `initialState` 需要在创建 hanzo 实例的时候传入，一般是 `{}`。前面已经说了，创建 reducer 的时候，就已经把默认值传给每个 reducer 了。这里就不需要再为 `initialState` 创建值了。

不过这段的重点应该是 `getReducer` 方法：

```javascript
function getReducer() {
  // extra reducers
  const extraReducers = plugin.get('extraReducers');

  const mergeReducers = deepmerge.all([this._reducers, extraReducers])
  for(let k in mergeReducers) {
    if(typeof mergeReducers[k] === 'object') {
      mergeReducers[k] = combineReducers(mergeReducers[k])
    }
  }

  const navInitialState = this._router.router.getStateForAction(
    this._router.router.getActionForPathAndParams(this._router.initialRouteName)
  );

  const navReducer = (state = navInitialState, action) => {
    const nextState = this._router.router.getStateForAction(action, state);

    // Simply return the original `state` if `nextState` is null or undefined.
    return nextState || state;
  };

  const appReducer = combineReducers({
    nav: navReducer,
    ...initialReducer,
    ...mergeReducers,
  });
  return appReducer
}
```

最终返回的 `appReducer` 中，包含了 router 的 reducer `navReducer` 还有用户的  `mergeReducers` 。

这里 `mergeReducers` 的生成非常有意思。之前我们保存在 hanzo 实例的 `_reducer`里的 reducer 结构是两层的：`_reducers.order.newOrder`. 但是 `combineReducer` 只能把一层的 reducer 转化为一个大的 reducer。因此，这里使用了两次 `combineReducer`。比如刚才的例子，第一次将 `order` 下的各个 reducer `combineReducer` 后得到一个 `order` 的 `mergeReducers['order']`。然后再把 `order` 同一层级的 reducer `combineReducer` 到 `appReducer` 下。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/redux_1.png?raw=true)

这样 reducer 的层次结构就和 state 一致了。

> **多层级的 state，可以多次使用 combineReducer 来达到将多层级 reducer 合并的目的。**

#### connect 相关(主要是设置 action)

前面已经成功隐藏了 `createStore`,`combineReducer`,`Provider`,`applymiddleware` 等方法，不过这还不够。对于用户来说，`dispatch(action)` 也没有必要让用户知道。使用者应该只需要提供一个方法，返回数据 `action.payload` 就行了，连 `action.type` 是什么用户都可以不必知道，由框架通过获取方法名推断得到。

所以需要使用 hanzo 提供的 `connect` 方法：

```javascript
module.exports.connect = function(state, model) {
  const actionCreators = {}
  let _handlers = []
  if(model) {
      if(model.handlers && Array.isArray(model.handlers)) {
        _handlers = _handlers.concat(model.handlers)
      }
      if(model.publicHandlers && Array.isArray(model.publicHandlers)) {
        _handlers = _handlers.concat(model.publicHandlers)
      }
  }
  if(_handlers.length > 0) {
    _handlers.map((key) => {
      if(key.action) {
        if(key.validate) {
          actionCreators[key.name] = createAction(model.namespace + '/' + key.name, key.action, key.validate)
        } else {
          actionCreators[key.name] = createAction(model.namespace + '/' + key.name, key.action)
        }
      } else if(key.handler) {
        let globalHandler = GlobalContext.getHandler(key.handler)
        if(globalHandler) {  
          if(key.validate) {
            actionCreators[key.name] = createAction(model.namespace + '/' + key.name, globalHandler, key.validate)
          } else {
            actionCreators[key.name] = createAction(model.namespace + '/' + key.name, globalHandler)
          }
        }
      } else {
        actionCreators[key] = createAction(model.namespace + '/' + key)
      }
    })
    const mapDispatchToProps = (dispatch) => bindActionCreators(actionCreators, dispatch)
    return connect(state, mapDispatchToProps)
  } else {
    return connect(state)
  }
}
```

关于 `mapStateToProps` 的部分我就不说了，主要看看 handler 如何变为 `mapDispatchToProps` 的过程。

hanlder 有三种用法：

```javascript
handlers: [
  'justAString',
  { name: 'handlerName', action: 'handlerActionFromActionJS' },
  { name: 'handlerName2', handler: 'handlerFromAnotherModule' }
]
```

第一种对应上面的 `else`:

```javascript
actionCreators[key] = createAction(model.namespace + '/' + key)
```

通过 redux-action 的 `createAction` 生成一个 **action 创建方法 createAction**，直接在方法名前加上 `namespace` 作为 `action.type` 传入。这样 `action.type` 就和前面处理 reducers 时添加 `namespace` 的做法一致了。调用这个 action 创建方法的话只需要传入一个 payload 就可以生成一个 action：

```javascript
let action = createAction(model.namspace + '/someaction')('payload')
```

第二种对应 `if(key.action)` 的情况：

```javascript
actionCreators[key.name] = createAction(model.namespace + '/' + key.name, key.action)
```

还是通过 `createAction`，不过这里**此时生成的 createAction 方法会在执行的时候接受参数，然后将其作为入参立刻执行 action 方法，并将 action 方法的返回值作为 payload 保存。**

> 虽然之前说了关于 `redux-thunk` 的 action 为 function 的情况。但是其实通过 `redux-actions` 创建的 action 其实不会是一个 action。这时候就要配合 `redux-promise-middleware` 了

`redux-promise-middleware` 就做了这样一个事情。它会检查 `action.payload` 调用后返回的是不是一个 promise。如果是一个 promise，它会在 `.then()` 方法中，替代使用者执行 dispatch 方法。它还能检查网络请求是成功还是失败，为原本的 `action.type` 添加相应的成功或者失败的后缀。

第三种对应 `else if(key.handler)` 的情况：

```javascript
let globalHandler = GlobalContext.getHandler(key.handler)
actionCreators[key.name] = createAction(model.namespace + '/' + key.name, globalHandler)
```

其实和第二种差不多，只是它不是从模块的 action 中获取方法，而是从 `globalHandler` 中找到对应的方法。

通过以上方法得到 `actionCreators` 之后，就要创建 `mapDispatchToProps` 了。由于使用者不需要知道 dispatch，所以这里使用 redux 提供的  `bindActionCreators`  方法，其实也没什么魔法，就是生成一个调用 dispatch 方法的方法：

```javascript
function bindActionCreator(actionCreator, dispatch) {
 	return (...args) => dispatch(actionCreator(...args))   
}
```

## Vuex 使用

Vue 提供了 Vuex 作为 redux 的解决方式。来看一下它的使用方式。

### 基本使用

#### store

```javascript
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default store = new Vuex.Store({
  state: {
    count: 0,
    todos: [
      { id: 1, text: '...', done: true },
      { id: 2, text: '...', done: false }
    ]
  },
  mutations: {
    increment (state) {
      state.count++
    }
  },
  actions: {
    increment (context) {
      context.commit('increment')
    }
  }
  getter: {
	doneTodos: state => {
      return state.todos.filter(todo => todo.done)
    }
  },

})
```

需要使用 `Vue.use(Vuex)` 像 Vue 注册插件 Vuex。**在 Vue 中，由于没有 react-redux 将属性传入展示组件，所以要在需要使用的地方 import 这个 store。**

#### state

通过 store 获取 state ：

```javascript
store.state.count		// -> 0
```

一般将 state 用在计算属性上：

```javascript
// 创建一个 Counter 组件
const Counter = {
  template: `<div>{{ count }}</div>`,
  computed: {
    count () {
      return store.state.count
    }
  }
}
```

#### getter

getter 用于简化 store 中 state 的计算：

```javascript
getters: {
  doneTodos: state => {
    return state.todos.filter(todo => todo.done)
  }
}
```

可以通过 `store.getters.xxx` 获取相关属性值。比如：

```javascript
store.getters.doneTodos // -> [{ id: 1, text: '...', done: true }]
```

#### mutation

根据 redux 所倡导的，改变 store 的唯一方式还是通过 dispatch 一个 action，而不是直接修改 store。Redux 中，reducer 是一个纯函数，用于修改 state。Vuex 也提供了相应方式 `mutation` 。

`mutation` 的写法非常类似 hanzo 中 `reducers` 的写法。mutation 中包含了很多的方法。方法接收 `initialState` 和一个值，即 `action.payload`。为什么不需要 `action.type` 了呢？因为方法名就是 type。比如下面的 `increment` 方法，就代表着这个 reducer 的 `action.type`

```javascript
mutations: {
  increment (state, n) {
    state.count += n
  }
}
```

不同的是，hanzo 中每个方法不能直接修改 state 值，要返回一个新的 state。而 `mutation` 的方法中直接修改 state。这种差异是由于 react 和 vue 响应式机制的不同而产生的。react 要通过比较 state 变化前后的不同，来找出需要重新渲染的组件。Vue 则是 state 的 set 方法通知页面中计算属性的改变。

触发方式上，Redux 中通过 `store.dispatch(action)` 依次触发所有 reducer。Vuex 中通过 `commit` 方法依次触发所有模块的 mutation，相当于把 `action` 拆成了 `action.type` 和 `action.payload` 分别传入：

```javascript
store.commit('increment', 10) 
```

这里与 hanzo 不同的是，hanzo 把与 commit 相对应的 dispatch 封装在 handler 中的相关方法里。而 Vuex 还需要自己调用。如果模块存在命名空间，那么写起来会比较冗长，而 hanzo 会自动为方法添加命名空间的前缀：

```javascript
store.commit('order/newOrder/increment', 10)
```

#### action

关于异步操作。Vuex 提供了action。

```javascript
actions: {
  incrementAsync (context, product) {
    setTimeout(() => {
      context.commit('increment', product.amount)
    }, 1000)
  }
}
```

其实还是相当于是在异步操作完了之后，触发一个 `mutation`。这里的 `context` 相当于当前模块的 store。即,只能拿到当前模块的 `commit` 方法和 `state`。

action 的触发使用 `dispatch` 方法。同样的，也是把 `action.type` 单独拿了出来：

```javascript
store.dispatch('incrementAsync', {
  amount: 10
})
```

vue 里通过 `dispatch` 和 `commit` 手动替代了 redux-thunk 的功能。

### 模块化

#### 基本使用

当工程变大后，所有状态和方法都写在 store 中是不明智的。所以 react 中才有了 hanzo 之类的框架。Vuex 提供了专门的模块化方式：

```javascript
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```

只创建一个 Store，把不同的模块以 module 的形式传入。通过 `store.state` 拿到去全局的 state，`store.state.a` 拿到相应模块的 state。

#### 模块的局部状态

模块化后模块内部的 `mutation` 和 `getter`。接收的第一个参数是模块的局部状态对象：

```javascript
const moduleA = {
  state: { count: 0 },
  mutations: {
    increment (state) {
      // 这里的 `state` 对象是模块的局部状态
      state.count++
    }
  },

  getters: {
    doubleCount (state) {
      return state.count * 2
    }
  }
}
```

对于模块内部的 action，局部状态通过 `context.state` 暴露，根节点状态为 `context.rootState`:

```javascript
const moduleA = {
  // ...
  actions: {
    incrementIfOddOnRootSum ({ state, commit, rootState }) {
      if ((state.count + rootState.count) % 2 === 1) {
        commit('increment')
      }
    }
  }
}
```

#### 命名空间

默认情况下，模块内部的 action、mutation 和 getter 是注册在**全局命名空间**的——这样使得多个模块能够对同一 mutation 或 action 作出响应。

可以通过添加 `namespaced: true` 的方式使其成为带命名空间的模块。当模块被注册后，它的所有 getter、action 及 mutation 都会自动根据模块注册的路径调整命名。例如：

```javascript
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,

      // 模块内容（module assets）
      state: { ... }, // 模块内的状态已经是嵌套的了，使用 `namespaced` 属性不会对其产生影响
      getters: {
        isAdmin () { ... } // -> getters['account/isAdmin']
      },
      actions: {
        login () { ... } // -> dispatch('account/login')
      },
      mutations: {
        login () { ... } // -> commit('account/login')
      },
    }
  }
})
```

### 总结

Vuex 在 reducer 部分的处理和 hanzo 非常相似，都是将 `action.type` 转化为一个个方法，都可以为每一个 `action.type` 的方法添加命名空间。都可以在每一个模块的 reducer 中获取到当前模块的 state。

但是在 dispatch 方法的几乎没有做太多的处理，还是需要使用者根据实际情况，去获取 store 调用 `store.dispatch` 或者 `store.commit` 方法，可以做进一步的封装，尤其是使用命名空间的情况下。















