title: NodeJs学习笔记
date: 2016/9/22 10:07:12  
categories: JavaScript
tags:
	- 学习笔记
	- NodeJs

---


<!--more-->

## 模块
### 实现
模块化的好处不多说，下面看看Node中如何模块化.我们编写了一个`hello.js`文件，这个`hello.js`文件就是一个模块，模块的名字就是文件名（去掉`.js`后缀），所以`hello.js`文件就是名为`hello`的模块:

```javascript
var s = 'Hello';

function greet(name) {
    console.log(s + ', ' + name + '!');
}

module.exports = greet;
```

上面定义了一个函数`greet()`，`module.exports = greet`的意思是，把函数greet作为模块的输出暴露出去，这样其他模块就可以使用greet函数了。

其他模块怎么使用`hello`模块的这个`greet`函数呢？我们再编写一个`main.js`文件，调用`hello`模块的`greet`函数：

```javascript
// 引入hello模块:
var greet = require('./hello');

var s = 'Michael';

greet(s); // Hello, Michael!
```

`greet`变量就是上面`module.exports = greet`输出的`greet`。在使用`require()`引入模块的时候，请注意模块的相对路径。

一个模块想要对外暴露变量（函数也是变量），可以用`module.exports = variable;`，一个模块要引用其他模块暴露的变量，用`var ref = require('module_name');`就拿到了引用模块的变量。

### 原理
模块化保证不同模块可以使用相同的变量名，那么这是如何实现的呢？

```javascript
// 准备module对象,require就是根据id获得相应module的exports:
var module = {
    id: 'hello',
    exports: {}
};
var load = function (module) {
    // 读取的hello.js代码:
    function greet(name) {
        console.log('Hello, ' + name + '!');
    }

    module.exports = greet;
    // hello.js代码结束
    return module.exports;
};
var exported = load(module);
// 保存module:
save(module, exported);
```

全局变量现在变成了匿名函数内部的局部变量，所以，Node利用JavaScript的函数式编程的特性，轻而易举地实现了模块的隔离。

变量`module`是Node在加载js文件前准备的一个变量，并将其传入加载函数，我们在`hello.js`中可以直接使用变量`module`原因就在于它实际上是函数的一个参数.

通过把参数`module`传递给`load()`函数，`hello.js`就顺利地把一个变量传递给了Node执行环境，Node会把`module`变量保存到某个地方。

由于Node保存了所有导入的`module`，当我们用`require()`获取`module`时，Node找到对应的`module`，把这个`module`的`exports`变量返回，这样，另一个模块就顺利拿到了模块的输出。


## 基本模块
### 基本模块
#### global
JavaScript有且仅有一个全局对象，在浏览器中，叫`window`对象。而在Node.js环境中，也有唯一的全局对象，叫`global`。

#### process
`process`也是Node.js提供的一个对象，它代表当前Node.js进程。通过`process`对象可以拿到许多有用信息：

```javascript
> process === global.process;
true
> process.version;
'v4.5.0'
> process.platform;
'darwin'
> process.arch;
'x64'
> process.cwd(); //返回当前工作目录
'/Users/michael'
> process.chdir('/private/tmp'); // 切换当前工作目录
undefined
> process.cwd();
'/private/tmp'
```

JavaScript程序是由事件驱动执行的单线程模型，Node.js也不例外。Node.js不断执行响应事件的JavaScript函数，直到没有任何响应事件的函数可以执行时，Node.js就退出了。如果我们想要在下一次事件响应中执行代码，可以调用`process.nextTick()`：

```javscripipt
// process.nextTick()将在下一轮事件循环中调用:
process.nextTick(function () {
    console.log('nextTick callback!');
});
console.log('nextTick was set!');
```

用Node执行上面的代码`node test.js`，你会看到，打印输出是：

```javascript
nextTick was set!
nextTick callback!
```

这说明传入`process.nextTick()`的函数不是立刻执行，而是要等到下一次事件循环。

Node.js进程本身的事件就由`process`对象来处理。如果我们响应`exit`事件，就可以在程序即将退出时执行某个回调函数：

```javascript
// 程序即将退出时的回调函数:
process.on('exit', function (code) {
    console.log('about to exit with code: ' + code);
});
```

#### 判断JavaScript执行环境
有很多JavaScript代码既能在浏览器中执行，也能在Node环境执行，但有些时候，程序本身需要判断自己到底是在什么环境下执行的，常用的方式就是根据浏览器和Node环境提供的全局变量名称来判断：

```javscript
if (typeof(window) === 'undefined') {
    console.log('node.js');
} else {
    console.log('browser');
}
```

### fs
#### 异步读取文件
```javascript
var fs = require('fs');

fs.readFile('sample.txt', 'utf-8', function (err, data) {
    if (err) {
        console.log(err);
    } else {
        console.log(data);
    }
});
```

`sample.txt`文件必须在当前目录下，且文件编码为`utf-8`。异步读取时，传入的回调函数接收两个参数，当正常读取时，`err`参数为`null`，`data`参数为读取到的`String`。当读取发生错误时，`err`参数代表一个错误对象，`data`为`undefined`。这也是Node.js标准的回调函数：第一个参数代表错误信息，第二个参数代表结果。

当读取二进制文件时，不传入文件编码时，回调函数的`data`参数将返回一个`Buffer`对象。在Node.js中，`Buffer`对象就是一个包含零个或任意个字节的数组（注意和`Array`不同）。

```javascript
var fs = require('fs');

fs.readFile('sample.png', function (err, data) {
    if (err) {
        console.log(err);
    } else {
        console.log(data);
        console.log(data.length + ' bytes');
    }
});
```

`Buffer`对象可以和`String`作转换，例如，把一个`Buffer`对象转换成`String`：

```javascript
// Buffer -> String
var text = data.toString('utf-8');
console.log(text);
```

或者把一个`String`转换成`Buffer`：

```javascript
// String -> Buffer
var buf = new Buffer(text, 'utf-8');
console.log(buf);
```

#### 同步读取文件


















