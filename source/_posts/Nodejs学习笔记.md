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






















