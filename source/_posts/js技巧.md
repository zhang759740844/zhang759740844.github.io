title: js技巧
date: 2018/7/31 14:07:12  
categories: JavaScript
tags: 
	- 持续更新
	
---

记录一些 js 使用技巧

<!--more-->

### && 的使用

平时我们条件判断的时候，通常是这样的：

```js
if (true) {
    console.log('true')
}
```

但是这样的写法并不简练，可以使用 `&&`，等效：

```javascript
true && console.log('true')
```

但是，要注意，由于不加 `;` 的缘故，不能写成 `(true) && console.log('true')`，因为在某些情况下会被错误解释：

```javascript
let a = someFunc()
(true) && console.log('true')
```

会被解释成 `let a = someFunc()(true) && console.log('true')`



### devDependencied 和 dependencies 区别

一般情况下，我们使用 `npm install module-name -save` 来把模块和版本号添加到 dependency 中，`npm install module-name -save-dev` 来把模块和版本号添加到 devdependencies 中。

这两个依赖的区别在于，在本地构建代码的时候需要用到 `devDependencied` 中的代码。比如我们需要使用 babel,eslint,webpeck 来构建代码，这些就会放在 `devDependencied` 中。这些依赖代码不会存在于构建好的项目代码中。

```
假设有以下两个模块：
模块A
- devDependencies
  模块B
- dependencies
  模块C
模块D
- devDependencies
  模块E
- dependencies
  模块A
npm install D的时候， 下载的模块为：
- D
- A
- C
当我们下载了模块D的源码，并且在根目录下npm install， 下载的模块为：
- A
- C
- E
```

### !! 的使用