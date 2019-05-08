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

通过反引号包裹的就是注入全局的样式，可以在全局任意位置引入：

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

