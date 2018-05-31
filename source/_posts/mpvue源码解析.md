title: mpvue源码解析
date: 2018/5/31 11:07:12
categories: MiniProgram
tags:
	- 源码解析
	
---

mpvue 是如何把 vue 转化成小程序可执行的代码，这是本篇的重点。

<!--more-->



## mpvue 初始化入口

小程序在启动的时候会执行 app.js 文件，这个文件里一般是执行小程序的 `App()` 方法，将一些 App 层级的回调方法提供给小程序。小程序启动后，立刻执行其中的 `onLaunch` 回调。mpvue 也需要遵循这样的规则，提供一个执行 `App()` 方法的 app.js 文件。

mpvue 通过 webpack 打包后生成了小程序所需的 app.js 。其中的玄机也都在这个 app.js 中。这个 app.js 引入了三个文件：

```javascript
require('./static/js/manifest')
require('./static/js/vendor')
require('./static/js/app')
```

第一个文件 manifest.js ，主要定义了一个 `webpackJsonCallback` 方法，然后将其存储到 `global["webpackJsonp"]` 中。这个方法接收并保存 webpack 生成的模块 `moreModules`，然后执行某些模块 `executeModules`：

```javascript
global["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) { ... }
```

第二个文件 vender.js，就直接调用了前面保存在 `global["webpackJsonp"]` 中的方法。 `moreModules` 包含所有非页面的模块，也就是 node_modules 中的各种模块：

```javascript
global.webpackJsonp([1],[/* 各种模块 */]);
```

这里只是把各个模块保存起来，还没有执行。执行在第三个文件 app.js 中，之后的重点也是这个 app.js。app.js 也是调用 `global["webpackJsonp"]` 的方法，不过它需要执行模块二：

```javascript
global.webpackJsonp([2],[/* 各个app级的模块 */],[2]);
```

## 小程序初始化——模块二

模块二代表着小程序的初始化过程。代码如下:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_1.png?raw=true)

其中主要是执行了这样的一个方法：

```javascript
(function(global) {
    ....
}.call(__webpack_exports__, __webpack_require__(3)))
```

### 小程序的全局变量

细看一下 `__webpack_require__(3)` 是个什么，上面说过非页面级的模块都在 vender.js 中定义，我们就在 vender.js 中找：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_2.png?raw=true)

这个模块的作用就是返回小程序的全局对象 `window`。

再回到二号模块，细看一下你就会发现，这就是 mpvue 项目中主目录下 `main.js` 在 webpack 打包后的代码。也就是说小程序在启动后会率先执行我们的 `main.js` 文件。 `window` 对象也会被作为参数 `global` 传入。如果我们想要和原生小程序一样使用 `getApp().globalData` 获取全局变量的话，我们可以设置：

```javascript
...
const app = new Vue(App)
app.$mount()
// 一定要在 app.$mount() 之后
global.getApp().globalData = {props: 'props'}
...
```

但是切记，一定要在 `app.$mount()` 之后，因为原生小程序的 `App()` 方法是在这个方法中调用的，这点后面会说。如果还没有执行 `app.$mount()`，那么 `global.getApp()` 获取到的就是 undefined。

### Vue 初始化

再回到模块二，main.js 中的 `import Vue from 'vue'` 被打包成了 `__WEBPACK_IMPORTED_MODULE_0_vue__` ，需要引入模块1：

```javascript
import Vue from 'vue'
=>
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0_vue__ = __webpack_require__(1);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0_vue___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_0_vue__);
```

模块一就是 node_module 中的 mpvue。这是一个 5000+ 行的长文件。它的作用就是设置 `Vue$3` 这个类对象。主要调用了几个方法，如下图所示，具体方法使用的时候再说：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_3.png?raw=true)

### App 初始化

Vue 初始化好之后，接着就是初始化 App。`import App from './App'` 被打包成了 `__WEBPACK_IMPORTED_MODULE_1__App__`，需要引入模块 4：

```javascript
import App from './App'
=>
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1__App__ = __webpack_require__(4);
```



