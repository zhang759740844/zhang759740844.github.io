
title: 《Secrets of the Javascript Ninja》 读书笔记
date: 2018/2/28 16:07:12  
categories: JavaScript
tags:
	- 学习笔记
---

这是一本不错的介绍 ES6 的书

<!--more-->

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SOJN.png?raw=true)



## 热身

### 创建页面

当浏览器获得服务器返回的 HTML，CSS 以及 JS 代码后，执行以下两步：

1. 解析并创建页面
2. 创建一个循环，等待事件发生

#### 创建页面

创建页面从上而下，循环进行以下两步：

1. 解析 HTML 页面成为 DOM
2. 执行 JS 代码

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SOJN_1.png?raw=true)

JS 代码由浏览器的 JS 引擎执行。浏览器暴露给 JS 引擎的一个最重要的对象是 `window`。`window` 是所有其他全局属性以及浏览器 API 的根。JS 中创建的全局变量(应该就是不用 var 声明的那种)，都被放到了 `window` 下。它最重要的一个全局属性是 `document`。这个属性被用来呈现当前页面的 DOM，因此能用它获取 DOM 中的任意的节点。`getElementById()` 这个方法是很有用的，获取了 DOM 的节点后，之后就可以动态的修改页面了。

```javascript
var first = document.getElementById("First");
```

JS 代码分为两种，一种是全局代码，一种是函数代码。函数代码在一个 `function` 为关键字的函数内。全局代码则在函数外。全局代码会被立刻执行，而函数代码必须要在被调用时才执行。下图中 `addmessage()` 方法用来给 DOM 的某个节点增加子节点，需要被调用才能执行：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SOJN_2.png?raw=true)

HTML 是由上到下解析的，中间遇到 JS 代码立刻执行，执行完了再继续解析。所以 JS 代码只能获取并操作在它之前出现的 DOM 的节点。因此一般将 JS 放在页面的底部。

#### 处理事件

浏览器是一个单线程模型，每次只能执行一个事件。因此，浏览器使用 *事件队列(event queue)* 来完成记录那些发生了但是还没有被处理的事件。浏览器检查事件队列中是否有事件。有的话选取第一个执行，其他的事件等待执行；没有的话继续检查。

需要注意的是，把事件放入事件队列的过程并不在我们说的那个单线程中执行。(这是显而易见的，不然在执行的时候的事件不就接收不到了么)

有两种方式添加事件回调，一种是将函数赋给一个特殊的实行，还有一种是使用 `addEventListener` 方法：

```javascript
// 赋值的方式
window.onload = function(){};
document.body.onclick = function(){};
```

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SOJN_3.png?raw=true)

## 理解函数

### 定义和参数

#### 函数与属性

函数能像对象一样被被传递或者被保存。比如下面的例子中，将函数当做对象 `store` 的一个键 `add` 的值。这个函数又接受一个入参为 `fn` 的函数。判断其是否有 `id` 属性，将其保存在 `cache` 的相应位置上：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SOJN_4.png?raw=true)

刚才说到函数也是拥有属性的，例如下面 `isPrime` 函数可以定义一个 `answers `属性：

```javascript
function isPrime() {
    if (!isPrime.answers) {
        isPrime.answers = 'old property';
    }
}

console.log(isPrime.answers);	// undefined
isPrime();
console.log(isPrime.answers);	// old property
isPrime.answers = 'new property';	
console.log(isPrime.answers);	// new property
console.log(isPrime.questions);	// undifined
isPrime.questions = 'new property';
console.log(isPrime.questions);	// new property
```

如上所示，为函数定义属性一定不要用 `this`，因为 `this` 指代的是调用者，而不是函数本身。函数本身的属性就直接用函数名来获取：`isPrime.answers`。如果在函数内部定义的属性，必须要在函数执行后，属性才会存在。这时候你就可以在外部改变属性的值了。另外，在函数外部也可以为函数添加属性。

> 函数属性还是挺有用的。有时候调用函数会用到的变量，如果不能用函数属性保存的话，我们就要在外部再创建一个变量，让函数获取。但是本身这个外部变量在外部并没有其它作用，就显得很多余。

#### 







 















