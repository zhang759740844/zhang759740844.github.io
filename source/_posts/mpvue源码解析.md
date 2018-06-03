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

## 小程序初始化

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

模块一就是 node_module 中的 mpvue。这是一个 5000+ 行的长文件。它的作用就是设置 `Vue$3` 这个类对象。主要调用了几个方法，如下图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_3.png?raw=true)

其中 `initGlobalAPI` 设置的这些属性是在 `Vue.construcotr` 上设置的，也就是图上黄色部分的属性。其余的方法都是在 `Vue.prototype` 上设置的。

需要注意的是设置中有一句代码：

```javascript
Vue.options._base = Vue;
```

它将 `Vue.constructor` 上的属性都设置到 `VUe.options._base` 下。

### App 初始化

Vue 初始化好之后，接着就是初始化 App。`import App from './App'` 被打包成了 `__WEBPACK_IMPORTED_MODULE_1__App__`，需要引入模块 4：

```javascript
import App from './App'
=>
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1__App__ = __webpack_require__(4);
```

模块四的代码如下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_4.png?raw=true)

需要引入模块0的 `normalizeComponent`，这个方法用来将 `<template>` 和 `<script>` 保存到一个对象中去。可以看到，打包后，vue 文件中的 `<template>` `<styles>` `<script>` 标签都拿出来了（`<script>` 标签就是图片中的模块7）。

模块4最终 export 的是  `normalizeComponent` 创建的，包含了 `<script>` 标签内各个属性方法的一个对象（还应该包含 `<template>` 标签下的对象的，但是 App 没有视图），如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_5.png?raw=true)

### 创建 Vue 实例

`new Vue(App)` 操作实则就是调用 init 方法。上面的 Vue 初始化图设置的属性和方法都在 `__proto__` 下，此处调用的 `_init` 方法为 Vue 实例初始化了很多属性，如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_6.png?raw=true)

其中 `$options` 下包含了模块4 export 的自己写的 `<script>` 下的对象以及 `Vue.option` 中的对象。App 级的 Vue 创建没有太多东西，大多数属性都是空的。

### 调用 $mount

前面把 Vue 都初始化完了，到这一步开始设置 MP 了。首先会为 Vue 设置一个 `$mp` 属性，之后关于小程序的方法属性都会在之后加到 `$mp` 之下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_8.png?raw=true)

mpvue 会调用了小程序的 `App()` 方法，写入我们在 `<script>` 标签中的各种回调：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_7.png?raw=true)

可以看到，`globalData` 是在此时初始化的，所以印证了上面说的全局变量要在 `$mount` 方法后设置。

另外，`onLaunch` 方法和 `onShow` 方法都会带一个 options 属性，包括一些参数啊场景值之类的。所以最好在 `onLaunch` `onLoad` 方法中初始化，这样就不用写冗长的 `this.$root.$mp.query.xxx` 获取参数了。

 `onLaunch`  方法中的 `this` 就是小程序提供的 `getApp()` 方法的返回值。mpvue 将其设置为 `$mp` 的 app 属性下，并把 `onLaunch` 的入参保存在了 `$mp` 的 `appOptions` 以及 `getApp()` 的 `globalData` 下。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_9.png?raw=true)

至此，小程序 App 级的初始化完成了。由于不涉及视图，所以整个流程非常简单。当调用 小程序的 `App()` 后，就是微信的操作了。WXService 在回调 `onLaunch` 等一些列方法后。然后就进入开始加载 Page 了。

## 页面初始化

微信会根据 `app.json` 中定义的 pages 数组的第一个元素作为首页显示。mpvue 会在打包生成 `app.json` 的时候将 `main.js` 中带有 `^` 的页面设置为 pages 数组的第一个元素。 `main.js` 中只要写一个首页信息就可以了。其余页面都是打包的时候，webpack 分析项目目录结构生成的。

Page 涉及到具体的视图的展示，会比 App 初始化复杂很多。先来看首页 page 的 `main.js`：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_10.png?raw=true)

这里虽然也叫 app，其实是初始化 page。`import vue` 其实都是一样的。直接看 `import App from './index'`

### page 初始化

同之前一样，也是通过 `normalizeComponent` 方法创建 Component。但是 Page 的 `<script>` 和 `<template>` 要比之前复杂很多。

`<script>` 标签中生成的代码如下:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_11.png?raw=true)

`components` 中会把子组件通过 `normalizeComponent` 创建的 Component 对象保存起来。

`<template>` 标签在打包后会根据 dom 结构生成一个 `render` 方法：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_12.png?raw=true)

其中 `_c` 就是 `$createElement` 方法，用于创建节点，这个后面再说。

最后生成的 page 中的属性方法如下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/mpvue_13.png?raw=true)

### 创建 Vue 实例

创建爱你 vue 主要就是创建了 computed 和 watch



### $mount

