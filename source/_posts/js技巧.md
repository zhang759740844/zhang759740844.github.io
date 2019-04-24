title: js技巧
date: 2018/7/31 14:07:12  
categories: JavaScript
tags: 

- 持续更新
  	

---

记录一些 js 使用技巧

<!--more-->

### 如何创建一个带有默认值的数组

```js
// 方法1
Array.apply(null, Array(30)).map(() => 4)
// 方法2
Array(30).fill(4)
```



### 判断是否是 promsie 类型

基本类型的判断可以通过 `typeof` 实现:

```javascript
typeof '123' === 'string'
```

promise 类型可以通过 `instanceof` 判断：

```javascript
new Promise() instanceof Promise
```



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

这两个依赖的区别在于，在本地构建代码的时候需要用到 `devDependencies` 中的代码。比如我们需要使用 babel,eslint,webpeck 来构建代码，这些就会放在 `devDependencies` 中。这些依赖代码不会存在于构建好的项目代码中。

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

> devdependency 中的文件如果使用 npm publish 发布 npm 包也不会被上传

### !! 的用法

很多时候，我们会看到这样的写法：

```javascript
if (!!a) {
    ...
}
```

这有什么用呢？其实这个等价于

```javascript
var a;
if (a != null && typeof(a) != undefined && a != '' && a != 0) {
    //a有内容才执行的代码  
}
```

其实就是当 a 不为空，不为undefined，不为空字符串，不为0的时候，即 a 有意义的时候才执行。!! 这种两次取反的操作简化了判断条件。

### 自动推测分号

JavaScript 提供了一种自动推测的方式使我们不主动添加分号。一般情况下我们使用的时候不会出错。但是要注意一种情况：

- **在以 (、[、+、- 或 / 字符开头的语句前绝不能省略分号。**

比如：

```javascript
let e = f
(a === b || c === d) && g()
```

第二句表示在 a 和 b 相等，或者 c 和 d 相等的情况下， 执行 g 方法。但是由于 `(` 在开头，此处会被推测为：

```javascript
let e = f(a === b || c === d) && g()
```

把 f 推测为了一个方法。