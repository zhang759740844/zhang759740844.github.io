title: react.js 使用
date: 2019/5/6 14:07:12  
categories: JavaScript
tags: 

 - 学习笔记
	
---

写了一年多的 RN，react 包括 redux 还是很熟练的，但是写起 pc 端顿时有点懵逼。

<!--more-->

## styled-components

当我们在一个 js 文件中引入一个 css 文件的时候，实际上是全局引用的。这样会造成样式混乱。需要引入样式模块。因此可以使用 styled-components

### 安装

```bash
npm install --save styled-components
```

现在样式不再使用 css 定义，而是使用 js 定义

###  基本使用

在 style.js 中定义组件，定义了一个叫 `HeaderWrapper` 的 div 标签：

```js
import styled from 'styled-components'

export const HeaderWrapper = styled.div`
	height: 54px;
	background: red;
`
```

现在可以把 `HeaderWrapper` 当成普通组件一样使用：

```jsx
import React, { Component } from 'react'
import {
  HeaderWrapper
} from './style'

class Header extends Component {
  render () {
    return (
    	<HeaderWrapper>header</HeaderWrapper>
    )
  }
}
```

### 其他使用

#### 全局样式

创建 style.js 文件。引入全局样式模块 `injectGlobal`:

```js
import { injectGlobal } from 'styled-components'

injectGlobal`
	body {
		margin: 0;
		padding: 0;
		background: green;
	}
`
```

通过反引号包裹的就是注入全局的样式，然后在根视图中引入：

```js
import './style.js'
```

#### 引入资源

比如需要在样式中设置背景图片，直接使用相对路径是无用的，需要引入配合 webpack 使用：

```js
import logoPic from '../../assets/logo.png'

export const Logo = styled.a`
	background: url(${logoPic})
`
```

通过 import 引入的资源文件需要通过 `${}` 这样的模板语法使用。

#### 某一个组件的特定样式

某一个组件可以根据传入的 `className` 设置不同的样式：

```js
export const NavItem = styled.div`
	&.left {
		float: left;
	}
	&.right {
		float: right;
	}
	&.active {
		background: #fff;
	}
	line-height: 56px;
	padding: 15px;
`
```

使用 `&` 表示当前组件的 `className`。使用方式如下：

```jsx
import {NavItem} from './style.js'

<NavItem className='left active'>左边样式</NavItem>
<NavItem className='right'>右边样式</NavItem>
```

#### 除style外的默认属性

定义组件的时候可以给组件设置默认的属性：

```js
export const Navsearch = styled.input.attrs({
  placeholder: '搜索'
})`
	width: 90px;
	height: 32px;
`
```

这是一个输入框组件，通过 `attrs` 设置默认占位符属性。

#### 父组件中设置子组件的样式

父组件中可以通过子组件的 `className` 定位子组件。比如有一个子组件的 `className` 为 iconfont：

```js
export const parent = styled.div`
	height: 100px;
	.iconfont {
		position: absolute;
		right: 0;
	}
`
```

注意，组件设置自己的样式是通过 `&.classname`，而设置子组件的样式是通过 `.classname` 完成的。



## 路由

### 安装

```bash
npm install --save react-router-dom
```

