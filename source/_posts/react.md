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



### 设置样式

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

#### 父组件中设置子组件的样式

父组件中可以通过子组件的 `className` 定位子组件。比如有一个子组件的 `className` 为 iconfont：

```js
export const Parent = styled.div`
	height: 100px;
	.iconfont {
		position: absolute;
		right: 0;
	}
`
```

注意，组件设置自己的样式是通过 `&.classname`，而设置子组件的样式是通过 `.classname` 完成的。

#### 父组件中设置子标签的样式

其实就和 less 一样，可以在父样式中设置子样式：

```js
export const Parent = styled.div`
	div {
		height: 20px;
	}	
`
```

设置 Parent 组件内部的 div 默认是 20px

### 其他使用

#### 引入资源

比如需要在样式中设置背景图片，直接使用相对路径是无用的，需要引入配合 webpack 使用：

```js
import logoPic from '../../assets/logo.png'

export const Logo = styled.a`
	background: url(${logoPic})
`
```

通过 import 引入的资源文件需要通过 `${}` 这样的模板语法使用。





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





## 路由

### 安装

```bash
npm install --save react-router-dom
```

## CSS 技巧

### 屏幕居中

先把左上角移动到屏幕中间，再把自身向左上角移动一半：

```css
.center {
	position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

### float 的使用

float 和 absolute 类似，可以将视图脱离于默认的排布，只不过 absolute 可以设置相对定位的元素，以及上下左右的位置。而 float 只能浮动到上下左右四个边。

### 设置一个视图的高度为屏幕高度减去一个特定高度

`vh` 单位会把屏幕分为 100 等分。所以整个屏幕的高度为 `100vh` 。减去一个特定像素的高度后，要使用 `calc` 方法计算：

```css
.fullscreen {
  height: calc(100vh - 80px)
}
```

### display 为 `block` `inline-block` `inline` 的区别

1、display：block将元素显示为块级元素，从而可以更好地操控元素的宽高，以及内外边距，每一个块级元素都是从新的一行开始。

2、display : inline将元素显示为行内元素，高度，行高以及底边距不可改变，高度就是内容文字或者图片的宽度，不可以改变。多个相邻的行内元素排在同一行里，知道页面一行排列不下，才会换新的一行。

3、display：inline-block看上去值名inline-block是一个混合产物，实际上确是如此，将元素显示为行内块状元素，设置该属性后，其他的行内块级元素会排列在同一行。比如我们li元素一个inline-block，使其既有block的宽度高度特性，又有inline的同行特性，在同一行内有不同高度内容的元素时，通常要设置对齐方式如vertical-align: top;来使元素顶部对齐。



## 构建技巧

### devDependencied 和 dependencies 区别

一般情况下，我们使用 `npm install module-name -save` 来把模块和版本号添加到 dependency 中，`npm install module-name -save-dev` 来把模块和版本号添加到 devdependencies 中。

这两个依赖的区别在于，在本地构建代码的时候需要用到 `devDependencies` 中的代码。比如我们需要使用 babel,eslint,webpeck 来构建代码，这些就会放在 `devDependencies` 中。这些依赖代码不会存在于构建好的项目代码中。

> devdependency 中的文件如果使用 npm publish 发布 npm 包也不会被上传

### npx 的使用

一般我们安装一些 nodejs 的命令行工具需要先全局安装，比如使用 `create-react-app` 这种脚手架就需要先:

```bash
> npm install create-react-app -g
> create-react-app my-app
```

事实上，我们没有必要使用全局安装的方式就可以解决，把它安装到某一个项目中，然后使用 npx 执行：

```bash
> npm install --save-dev create-react-app
> npx create-react-app my-app
```

其实就相当于执行了 node-modules 下 bin 目录中的二进制文件：

```bash
> node-modules/.bin/create-react-app my-app
```

### 实践技巧

#### 请求添加 cookie

axios 默认的请求是不会添加 cookie 的，需要在请求的 option 字段中添加 `withCredentials: true` 这个选项才行。

这个选项要求请求不能跨域，或者跨域的域名在服务端 `access-control-allow-origin` 允许的域名中。

#### 设置 cookie

要想服务端通过 `set-cookie`能够成功设置 cookie 的前提是**请求不能跨域**名。跨域名的请求是无法设置 cookie 的。

之所以说是不能跨域名是因为 **cookie 是跨端口共享**的，即如果你的主站是 `localhost:3000` 请求的端口是 `localhost:3001`，那么 set-cookie 还是能够成功设置的。

> 简单来说，跨域请求是不能读取 cookie 的。但是跨端口设置 cookie 是能成功的。

另外要说明一下， 使用 `localhost` 和本机 ip 这是两个不同的域名，一定是跨域的。所以要设置 cookie 的时候，不要主站用 localhost，请求用 ip，或者主站用 ip，请求用 localhost。