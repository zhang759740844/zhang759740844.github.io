title: 小程序 typescript 实践
date: 2019/7/12 14:07:12  
categories: JavaScript
tags: 

 - 小程序
	
---

新的小程序打算用 ts 写，暂时踩了一些坑

<!--more-->

## 实践

### 编译ts的时候小程序控制台报错

刚开始实践的时候，小程序控制台一直报错，但是也没有具体的显示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ts_1.png?raw=true)

遇到这个错误非常头疼，虽然显示没有问题，但是每次编译都会报这个错让人很难受。之后进过一阵摸索后，发现对于 ts 的小程序项目，在点击编译后会执行 `npm run tsc` 进行将 ts 到 js 的转换：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ts_2.png?raw=true)

因此，我尝试手动在控制台中执行 `npm run tsc`，终于得到了错误的定位信息:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/ts_3.png?raw=true)

### 小程序hot reload

js 的小程序项目中 js 的变更会引起小程序的热重载，但是在 ts 的项目中就必须手动点击编译。这样的操作非常不友好。因此，我想自己实现 hot reload。

hot reload 的 node 库可以使用：chokidar

```bash
npm install --save-dev chokidar
```

然后在根目录创建 `hotreload.js` 文件，方案是监听到文件变更后触发 `npm run tsc`：

```js
const chokidar = require('chokidar')
const shell = require('shelljs')
chokidar.watch('./miniprogram', { cwd: '.' }).on('change', (event, path) => {
  shell.exec('npm run tsc')
  console.log('刷新')
})
```

上面就是 watch 的 `miniprogram` 目录下的所有文件的变更。但是这样还有问题。当 ts 变更后触发 `npm run tsc`，而 `npm run tsc` 更新 js 后又会重新触发 `npm run tsc`，变成了死循环。因此，要排除 js 文件的跟新：

```js
const chokidar = require('chokidar')
const shell = require('shelljs')
chokidar.watch('./miniprogram', { cwd: '.', ignored: /.js$/ }).on('change', (event, path) => {
  shell.exec('npm run tsc')
  console.log('刷新')
})
```















































