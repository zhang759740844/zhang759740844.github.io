title: JavaScript基本语法
date: 2016/9/9 10:07:12  
categories: JavaScript
tags:
	- 学习笔记
---

参考自[廖雪峰的JavaScript教程](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)

<!--more-->

## 基础语法
### 数据类型和变量
#### Number
JS不区分整数浮点数，统一用`Number `表示：
```javascript
1.2345e3; // 科学计数法表示1.2345x1000，等同于1234.5
NaN; // NaN表示Not a Number，当无法计算结果时用NaN表示
Infinity; // Infinity表示无限大，当数值超过了JavaScript的Number所能表示的最大值时，就表示为Infinity
```

`Number`可以直接运算：
```javascript
(1 + 2) * 5 / 2; // 7.5
2 / 0; // Infinity
0 / 0; // NaN
10 % 3; // 1 (取余)
10.5 % 3; // 1.5
```

#### 字符串
以单引号`'`或双引号`"`括起来的任意文本.如`'abc'` 

#### 布尔值
用`true`和`false`表示。

#### 比较运算符
JavaScript在设计时，有两种比较运算符：
第一种是`==`比较，它会自动转换数据类型再比较.
第二种是`===`比较，它不会自动转换数据类型，如果数据类型不一致，返回false，如果一致，再比较。

如果是比较类对象，就是比较两者的地址。如果地址不同，`==` 和 `===` 都返回 false。

`NaN`这个特殊的`Number`与所有其他值都不相等，包括它自己：
```javascript
NaN == NaN; // false
NaN === NaN; // false
```
唯一能判断`NaN`的方法是通过`isNaN()`函数：
```javascript
isNaN(NaN); // true
```

浮点数在运算过程中会产生误差，因为计算机无法精确表示无限循环小数。要比较两个浮点数是否相等，只能计算它们之差的绝对值，看是否小于某个阈值：
```javascript
Math.abs(1 / 3 - (1 - 2 / 3)) < 0.0000001; // true
```

#### 数组
JavaScript的数组可以包括任意数据类型。例如：
```javascript
[1, 2, 3.14, 'Hello', null, true];
```

另一种创建数组的方法是通过Array()函数实现：
```javascript
new Array(1, 2, 3); // 创建了数组[1, 2, 3]
```
然而，出于代码的可读性考虑，强烈建议直接使用`[]`。

数组的元素可以通过索引来访问。请注意，索引的起始值为0：
```javascript
var arr = [1, 2, 3.14, 'Hello', null, true];
arr[0]; // 返回索引为0的元素，即1
arr[5]; // 返回索引为5的元素，即true
arr[6]; // 索引超出了范围，返回undefined
```

#### 对象
JavaScript的对象是一组由键-值组成的无序集合，例如：
```javascript
var person = {
    name: 'Bob',
    age: 20,
    tags: ['js', 'web', 'mobile'],
    city: 'Beijing',
    hasCar: true,
    zipcode: null
};
```
JavaScript对象的键都是**字符串类型**，值可以是任意数据类型。

要获取一个对象的属性，我们用对象变量.属性名的方式：
```javascript
person.name; // 'Bob'
person.zipcode; // null
```

#### 变量与常量
ES6 之前，变量用 `var` 修饰，但是会引起问题，包括作用域问题与变量提升问题。使用 `let` 代替。
```javascript
let b = 1; // 申明了变量$b，同时给$b赋值，此时$b的值为1
```
ES6 之后，常量用 `const` 修饰，表示不会再改变的值：

```javascript
const b = '234'
```

> Swift 中变量为 `var` 常量为 `let` 略有不同。

### 字符串

#### 模板字符串
要把多个字符串连接起来，可以用`+`号连接：
```javascript
var name = '小明';
var age = 20;
var message = '你好, ' + name + ', 你今年' + age + '岁了!';
alert(message);
```

如果有很多变量需要连接，用`+`号就比较麻烦。ES6新增了一种模板字符串，表示方法和上面的多行字符串一样，但是它会自动替换字符串中的变量：
```javascript
var name = '小明';
var age = 20;
var message = `你好, ${name}, 你今年${age}岁了!`;
alert(message);
```

> 注意这里不只是 `${}` 外边的不再是单引号了，而是 ` `` `。

> 在 Swift 中有类似的模板字符串：`\(要打印的对象)`，类似于这里的`${}`。不过 Swift 写起来比这个好简单好记多了。 

#### 操作字符串

字符串常见的操作如下：
```javascript
var s = 'Hello, world!';
s.length; // 13
s[7]; // 'w'
s[12]; // '!'
```

字符串相当于是一个不能改变的数组。

#### indexOf
`indexOf()`会搜索指定字符串出现的位置:
```javascript
var s = 'hello, world';
s.indexOf('world'); // 返回7
s.indexOf('World'); // 没有找到指定的子串，返回-1
```

#### substring
`substring()`返回指定索引区间的子串：
```javascript
var s = 'hello, world'
s.substring(0, 5); // 从索引0开始到5（不包括5），返回'hello'
s.substring(7); // 从索引7开始到结束，返回'world'
```

### 数组
要取得`Array`的长度，直接访问`length`属性：
```javascript
var arr = [1, 2, 3.14, 'Hello', null, true];
arr.length; // 6
```

#### indexOf
与`String`类似，`Array`也可以通过`indexOf()`来搜索一个指定的元素的位置：
```javascript
var arr = [10, 20, '30', 'xyz'];
arr.indexOf(10); // 元素10的索引为0
arr.indexOf(20); // 元素20的索引为1
arr.indexOf(30); // 元素30没有找到，返回-1
arr.indexOf('30'); // 元素'30'的索引为2
```

#### slice
`slice()`就是对应`String`的`substring()`版本，它截取`Array`的部分元素，然后返回一个新的`Array`：
```javascript
var arr = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
arr.slice(0, 3); // 从索引0开始，到索引3结束，但不包括索引3: ['A', 'B', 'C']
arr.slice(3); // 从索引3开始到结束: ['D', 'E', 'F', 'G']
```

如果不给`slice()`传递任何参数，它就会从头到尾截取所有元素。利用这一点，我们可以很容易地复制一个`Array`



### 对象
```javascript
var xiaohong = {
    name: '小红',
    'middle-school': 'No.1 Middle School'
};
```
`xiaohong`的属性名`middle-school`不是一个有效的变量，就需要用`''`括起来。访问这个属性也无法使用`.`操作符，必须用`['xxx']`来访问：
```javascript
xiaohong['middle-school']; // 'No.1 Middle School'
xiaohong['name']; // '小红'
xiaohong.name; // '小红'
```

如果属性名是其他对象，那么需要用`[]`将其括起来，表示对外部对象执行`toString()`操作：
```javascript
var weight = function () {
    return "1weight"
};

var xiaoming = {
    name :  "小明",
    [weight()] :"属性名是返回值1weight",
    [weight]:"属性名是整个function"
};

console.log(xiaoming)
// 返回
{ name: '小明',
  '1weight': '属性名是返回值1weight',
  'function () {\n    return "1weight"\n}': '属性名是整个function' }
```

由于JavaScript的对象是动态类型，你可以自由地给一个对象添加或删除属性：
```javascript
var xiaoming = {
    name: '小明'
};
xiaoming.age; // undefined
xiaoming.age = 18; // 新增一个age属性
xiaoming.age; // 18
delete xiaoming.age; // 删除age属性
xiaoming.age; // undefined
delete xiaoming['name']; // 删除name属性
xiaoming.name; // undefined
delete xiaoming.school; // 删除一个不存在的school属性也不会报错
```

如果我们要检测`xiaoming`是否拥有某一属性，可以用`in`操作符,不过要小心，如果`in`判断一个属性存在，这个属性不一定是`xiaoming`的，它可能是`xiaoming`继承得到的：
```javascript
'name' in xiaoming; // true
'toString' in xiaoming; // true
```
要判断一个属性是否是`xiaoming`自身拥有的，而不是继承得到的，可以用`hasOwnProperty()`方法：
```javascript
var xiaoming = {
    name: '小明'
};
xiaoming.hasOwnProperty('name'); // true
xiaoming.hasOwnProperty('toString'); // false
```

### Map和Set
JS中默认对象表达方式`{}`可以视为其他语言中的`Map`或`Dictionary`的数据结构，即一组键值对。
但是JavaScript的对象有个小问题，就是**键必须是字符串**。但实际上Number或者其他数据类型作为键也是非常合理的。
为了解决这个问题，最新的ES6规范引入了新的数据类型`Map`。

#### Map
```javascript
var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);
m.delete('Adam'); // 删除key 'Adam'
m.get('Michael'); // 95
m.set('Zachary', 100);
```

#### Set
重复元素在Set中自动被过滤：
```javascript
var s = new Set([1, 2, 3, 3, '3']);
s; // Set {1, 2, 3, "3"}
```
注意数字`3`和字符串`'3'`是不同的元素。

通过`add(key)`方法可以添加元素到`Set`中，可以重复添加，但不会有效果：
```javascript
>>> s.add(4)
>>> s
{1, 2, 3, 4}
>>> s.add(4)
>>> s
{1, 2, 3, 4}
```
通过delete(key)方法可以删除元素：
```javascript
var s = new Set([1, 2, 3]);
s; // Set {1, 2, 3}
s.delete(3);
s; // Set {1, 2}
```

### iterable
ES6标准引入了新的`iterable`类型，`Array`、`Map`和`Set`都属于`iterable`类型。
具有`iterable`类型的集合可以通过新的`for ... of`循环来遍历。

用`for ... of`循环遍历集合，用法如下：
```javascript
var a = ['A', 'B', 'C'];
var s = new Set(['A', 'B', 'C']);
var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
for (var x of a) { // 遍历Array
    alert(x);
}
for (var x of s) { // 遍历Set
    alert(x);
}
for (var x of m) { // 遍历Map
    alert(x[0] + '=' + x[1]);
}
```

**`for ... of`循环和`for ... in`循环有何区别？**
`for ... in`实际上是**对象的属性名称(注意，是键，不是值)**。不要用在`Array`,`Set`,`Map`上，会出现奇怪的问题。
`for ... of`循环则完全修复了这些问题，它**只循环得到集合内该出现的元素(注意，是值，不是键)**，例如：
```javascript
var a = ['A', 'B', 'C'];
a.name = 'Hello';
a['4'] = 'D';
console.log(a);
for (var num of a){
    console.log(num)
}
输出：
[ 'A', 'B', 'C', , 'D', name: 'Hello' ]
A
B
C
undefined
D
```
`name`由于不是正常`Array`该有的，所以在`for ... of`循环时，其对应的值不会被输出。

然而，更好的方式是直接使用`iterable`内置的`forEach`方法，它接收一个函数，每次迭代就自动回调该函数。以`Array`为例：
```javascript
var a = ['A', 'B', 'C'];
a.forEach(function (element, index, array) {
    // element: 指向当前元素的值
    // index: 指向当前索引
    // array: 指向Array对象本身
    alert(element);
});
```
对于`Set`来说，由于没有索引，`index`也是指当前元素。
对于`Map`来说，`element`和`index`分别对应`Value`和`Key`.

## 函数
### 函数定义和调用
#### 定义函数
方式一：
```javascript
function abs(x) {
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
}
```
由于JavaScript的函数也是一个对象，上述定义的`abs()`函数实际上是一个函数对象，而函数名`abs`可以视为指向该函数的变量。

方式二：
```javascript
var abs = function (x) {
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
};
```
在这种方式下，`function (x) { ... }`是一个匿名函数，它没有函数名。但是，这个匿名函数赋值给了变量`abs`，所以，通过变量`abs`就可以调用该函数。

上述两种定义完全等价，注意第二种方式按照完整语法需要在函数体末尾加一个`;`，表示赋值语句结束。

> Swift 中使用 func 定义函数

#### 调用函数
由于JavaScript允许传入任意个参数而不影响调用，因此传入的参数比定义的参数多也没有问题，虽然函数内部并不需要这些参数：
```javascript
abs(10, 'blablabla'); // 返回10
abs(-9, 'haha', 'hehe', null); // 返回9
```

传入的参数比定义的少也没有问题：
```javascript
abs(); // 返回NaN
```
此时`abs(x)`函数的参数`x`将收到`undefined`，计算结果为`NaN`。

要避免收到undefined，可以对参数进行检查：
```javascript
function abs(x) {
    if ((typeof x) !== 'number') {
        throw 'Not a number';
    }
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
}
```

#### arguments
JavaScript还有一个免费赠送的关键字`arguments`，它只在函数内部起作用，并且永远指向当前函数的调用者传入的所有参数。`arguments`类似`Array`但它不是一个`Array`：
```javascript
function foo(x) {
    alert(x); // 10
    for (var i=0; i<arguments.length; i++) {
        alert(arguments[i]); // 10, 20, 30
    }
}
foo(10, 20, 30);
```
利用`arguments`，你可以获得调用者传入的所有参数。也就是说，即使函数不定义任何参数，还是可以拿到参数的值.实际上arguments最常用于判断传入参数的个数。

#### rest参数
由于JavaScript函数允许接收任意个参数，于是我们就不得不用`arguments`来获取所有参数.
ES6标准引入了`rest`参数，可以将剩余的参数放在`rest`中：
```javascript
function foo(a, b, ...rest) {
    console.log('a = ' + a);
    console.log('b = ' + b);
    console.log(rest);
}

foo(1, 2, 3, 4, 5);
// 结果:
// a = 1
// b = 2
// Array [ 3, 4, 5 ]

foo(1);
// 结果:
// a = 1
// b = undefined
// Array []
```

`rest`参数只能写在最后，前面用`...`标识，从运行结果可知，传入的参数先绑定`a`、`b`，多余的参数以数组形式交给变量`rest`，所以，不再需要`arguments`我们就获取了全部参数。
如果传入的参数连正常定义的参数都没填满，也不要紧，`rest`参数会接收一个空数组（注意不是`undefined`）。



### 方法
绑定到对象上的函数称为方法:
```javascript
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

xiaoming.age; // function xiaoming.age()
xiaoming.age(); // 今年调用是25,明年调用就变成26了
```

> 把 age 想象成 OC 中的一个 block 就好理解了。

和普通函数也没啥区别，但是它在内部使用了一个`this`关键字.
在一个方法内部，`this`是一个特殊变量，**它始终指向当前对象**，也就是`xiaoming`这个变量。所以，`this.birth`可以拿到`xiaoming`的`birth`属性。

让我们拆开写：
```javascript
function getAge() {
    var y = new Date().getFullYear();
    return y - this.birth;
}

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: getAge
};

xiaoming.age(); // 25, 正常结果
getAge(); // NaN
```

单独调用函数`getAge()`怎么返回了`NaN`？请注意，我们已经进入到了JavaScript的一个大坑里。JavaScript的函数内部如果调用了`this`，那么这个`this`到底指向谁？答案是，视情况而定！（如果是在浏览器中执行，那就是 `window`）如果以对象的方法形式调用，比如`xiaoming.age()`，该函数的`this`指向被调用的对象，也就是`xiaoming`，这是符合我们预期的。如果单独调用函数，比如`getAge()`，此时，该函数的`this`指向**全局对象**.

更坑爹的是，如果这么写：
```javascript
var fn = xiaoming.age; // 这个表示拿到xiaoming的age函数，而不是拿到上下文
fn(); // NaN
```
也是不行的！要保证`this`指向正确，**必须要给出明确的上下文对象**，必须用`obj.xxx()`的形式调用！谁调用，`this`就是指谁。上面仅仅相当于将`age`方法赋给`fn`.

>   在 ES6 之前，function 也可以通过 `new getAge()` 的方式直接创建对象。这样 this 就指向 new 出来的对象。不过 ES6 之后创建对象都使用 class 了，function 仅用来表示函数。
>
>   另外，function 中也能创建 function 的属性，你可以把它理解成闭包中保存的变量值。它的作用以及和 this 的区别在另一篇中介绍。

如果是这种情况：
```javascript
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - this.birth;
        }
        return getAgeFromBirth();
    }
};

xiaoming.age(); 
```

> js中允许，函数的嵌套，但是外部无法直接获取嵌套的函数，需要函数返回。**不要以为函数也有属性就把函数当成对象一样**。直接 `xiaoming.age.getAgeFromBirth()` 这是不行的。

又不对了。原因是`this`指针只在`age`方法的函数内指向`xiaoming`，在函数内部定义的函数，`this`又指向`undefined`了！需要这样修改：

```javascript
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var that = this; // 在方法内部一开始就捕获this
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - that.birth; // 用that而不是this
        }
        return getAgeFromBirth();
    }
};

xiaoming.age(); // 25
```

用`var that = this`;将`age`中的`this`捕获，就可以放心地在方法内部定义其他函数，而不是把所有语句都堆到一个方法中。

> 我觉得可以这么理解帮助记忆：每个方法里都有一个上下文context，如果用点语法的方式调用，那么就会自动将调用者设置为了上下文 context，然后就用 this 指针获取这个上下文 context。
>
> 上面的示例中，`return getAgeFromBirth()` 没有用点语法调用，也就是没有设置 `getAgeFromBirth` 方法的上下文，那么方法中**指向上下文的** `this` 指针就是 null，所以必须要在外部再重新声明一个变量，相当于手动获取一下外部的上下文。

#### apply
我们还是可以控制`this`的指向的！

要指定函数的`this`指向哪个对象，可以用函数本身的`apply`方法，它接收两个参数，第一个参数就是需要绑定的`this`变量，第二个参数是`Array`，表示函数本身的参数。

用`apply`修复`getAge()`调用:
```javascript
function getAge() {
    var y = new Date().getFullYear();
    return y - this.birth;
}

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: getAge
};

xiaoming.age(); // 25
getAge.apply(xiaoming, []); // 25, this指向xiaoming, 参数为空
```

再举一个例子：

```javascript
var numbers = {  
   numberA: 5,
   numberB: 10,
   sum: function() {
     console.log(this === numbers); // => true
     function calculate() {
       // this is window or undefined in strict mode
       console.log(this === numbers); // => false
       return this.numberA + this.numberB;
     }
     return calculate();
   }
};
numbers.sum(); // => NaN or throws TypeError in strict mode  

var numbers = {  
   numberA: 5,
   numberB: 10,
   sum: function() {
     console.log(this === numbers); // => true
     function calculate() {
       console.log(this === numbers); // => true
       return this.numberA + this.numberB;
     }
     // use .call() method to modify the context
     return calculate.call(this);
   }
};
numbers.sum(); // => 15

```


上面的`call()`与`apply()`是类似的方法，唯一区别是：
- `apply()`把参数打包成`Array`再传入；
- `call()`把参数按顺序传入。 

比如调用`Math.max(3, 5, 4)`，分别用`apply()`和`call()`实现如下：
```javascript
Math.max.apply(null, [3, 5, 4]); // 5
Math.max.call(null, 3, 5, 4); // 5
```
对普通函数调用，我们通常把this绑定为null。

#### .bind()

对比方法 `.apply()` 和 `.call()`，它俩都立即执行了函数，而 `.bind()` 函数返回了一个新方法，绑定了预先指定好的 `this` ，并可以延后调用。`.bind()` 方法的作用是创建一个新的函数，执行时的上下文环境为 `.bind()` 传递的第一个参数，它允许创建预先设置好 `this` 的函数。

```javascript
var numbers = {  
  array: [3, 5, 10],
  getNumbers: function() {
    return this.array;    
  }
};
// Create a bound function
var boundGetNumbers = numbers.getNumbers.bind(numbers);  
boundGetNumbers(); // => [3, 5, 10]  
// Extract method from object
var simpleGetNumbers = numbers.getNumbers;  
simpleGetNumbers(); // => undefined or throws an error in strict mode  
```

使用 `.bind()` 时应该注意，`.bind()` 创建了一个永恒的上下文链并不可修改。一个绑定函数即使使用 `.call()` 或者 `.apply()`传入其他不同的上下文环境，也不会更改它之前连接的上下文环境，重新绑定也不会起任何作用。


#### 装饰器
利用`apply()`，我们还可以动态改变函数的行为。

JavaScript的所有对象都是动态的，即使内置的函数，我们也可以**重新指向新的函数**。

现在假定我们想统计一下代码一共调用了多少次`parseInt()`，可以把所有的调用都找出来，然后手动加上`count += 1`，不过这样做太傻了。最佳方案是用我们自己的函数替换掉默认的`parseInt()`：
```javascript
var count = 0;
var oldParseInt = parseInt; // 保存原函数

window.parseInt = function () {
    count += 1;
    return oldParseInt.apply(null, arguments); // 调用原函数
};

// 测试:
parseInt('10');
parseInt('20');
parseInt('30');
count; // 3
```

### 高阶函数
一个可以接收另一个函数作为参的函数，称为高阶函数。
#### map
`map()`方法定义在JavaScript的`Array`中，我们调用`Array`的`map()`方法，传入我们自己的函数，就得到了一个新的`Array`作为结果：
```javascript
function pow(x) {
    return x * x;
}

var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
```

`map()`将传入的函数一一作用在数组的每一个元素上。

#### reduce
`Array`的`reduce()`把一个函数作用在这个`Array`的`[x1, x2, x3...]`上，这个函数必须接收两个参数，`reduce()`把结果继续和序列的下一个元素做累积计算，其效果就是：
```javascipt
[x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)
```

比如对`Array`求和：
```javascript
var arr = [1, 3, 5, 7, 9];
arr.reduce(function (x, y) {
    return x + y;
}); // 25
```

#### filter
用于把`Array`的某些元素过滤掉，然后返回剩下的元素
`Array`的`filter()`接收一个函数,把传入的函数依次作用于每个元素，然后根据返回值是`true`还是`false`决定保留还是丢弃该元素。

例如，在一个`Array`中，删掉偶数，只保留奇数，可以这么写：
```javascript
var arr = [1, 2, 4, 5, 6, 9, 10, 15];
var r = arr.filter(function (x) {
    return x % 2 !== 0;
});
r; // [1, 5, 9, 15]
```

#### sort
可以接收一个比较函数来实现自定义的排序
要按数字大小排序，我们可以这么写：
```javascript
var arr = [10, 20, 1, 2];
arr.sort(function (x, y) {
    if (x < y) {
        return -1;
    }
    if (x > y) {
        return 1;
    }
    return 0;
}); // [1, 2, 10, 20]
```

**注意**：`sort()`方法会直接对`Array`进行修改，它返回的结果仍是当前`Array`

### 闭包
#### 函数作为返回值
高阶函数除了可以接受函数作为参数外，还可以把函数作为结果值返回。

比如返回一个求和的函数：
```javascript
function lazy_sum(arr) {
    var sum = function () {
        return arr.reduce(function (x, y) {
            return x + y;
        });
    }
    return sum;
}
```
当我们调用`lazy_sum()`时，返回的并不是求和结果，而是求和函数：
```javascript
var f = lazy_sum([1, 2, 3, 4, 5]); // function sum()
```
调用函数f时，才真正计算求和的结果：
```javascript
f(); // 15
```
当`lazy_sum`返回函数`sum`时，相关参数和变量都保存在返回的函数中，这种称为“闭包（Closure）”(但是JS中`this`不遵循闭包)

#### 闭包
需要注意的问题是，返回的函数并没有立刻执行，而是直到调用了f()才执行。我们来看一个例子：
```javascript
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push(function () {
            return i * i;
        });
    }
    return arr;
}

var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];
```

上面的例子中，每次循环，都创建了一个新的函数，然后，把创建的3个函数都添加到一个Array中返回了，**或者说返回了三个闭包**。调用的结果全是`16`,因为返回函数都引用了变量`i`，由于闭包性，等到3个函数都返回时，它们所引用的变量i已经变成了`4`。

**这是因为 JS 中的闭包对外部变量都是引用传递。**

> 这和 iOS 中的闭包不同，iOS 中会捕获外部的局部变量，也就是保存值。所以 i 是几就是几，而 js 中应该只是获得了 i 的地址。

那么怎么修改呢？有两种方法，**一种方法是将引用传递变为值传递。还有一种是不要引用同一个变量**。

**方法一**是外部再创建一个函数，由传参的方式把变量传入。这样就是将引用传递变为值传递了。无论该循环变量后续如何更改，已绑定到函数参数的值不变：

```javascript
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push((function (n) {
            return function () {
                return n * n;
            }
        })(i));
    }
    return arr;
}

var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];

f1(); // 1
f2(); // 4
f3(); // 9
```

注意这里用了一个“创建一个匿名函数并立刻执行”的语法：
```javascript
(function (x) {
    return x * x;
})(3); // 9
```

**实际上，上面的代码相当于在外层函数拿了一个变量`n`截取了`i`,由于外层函数是立刻执行了的匿名函数(即立刻执行了`var n = i;`)，那么`n`就固定了下来**：
```javascript
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push((function () {
            var n =i
            return function () {
                return n*n;
            }
        })());
    }
    return arr;
}
```


> 这里就相当于每个循环都创建了一个独立的 n，上面呢是公用一个 i

更直接的方式就是**方法二**，用 `let` 代替 `var`。因为 `var` 的作用域是函数，所以，循环期间 `i` 是不会回收的。但是如果是 `let`，那么作用域就是循环的循环体。也就是每次循环体执行，都会创建一个新的 `i`，而不是像 `var` 一样公用一个.

```javascript
function count() {
    var arr = [];
    for (let i=1; i<=3; i++) {
        arr.push((function () {
            return function () {
                return i*i;
            }
        })());
    }
    return arr;
}
```

### 箭头函数
ES6标准新增了一种新的函数：Arrow Function（箭头函数）。
```javascript
x => x*x
```
相当于一个输入为`x`输出为`x*x`的匿名函数
```
function (x) {
    return x * x;
}
```
如果参数不是一个，就需要用括号()括起来：
```javascript
// 两个参数:
(x, y) => x * x + y * y

// 无参数:
() => 3.14

// 可变参数:
(x, y, ...rest) => {
    var i, sum = x + y;
    for (i=0; i<rest.length; i++) {
        sum += rest[i];
    }
    return sum;
}
```
语法糖，类似于python中的lambda函数

当然，箭头函数还是有点用处的，由于是es6的新特性，**箭头函数内部的this是词法作用域**，由上下文确定。
试做比较：
```javascript
//由于JavaScript函数对this绑定的错误处理，下面的例子无法得到预期结果
var obj = {
    birth: 1990,
    getAge: function () {
        var b = this.birth; // 1990
        var fn = function () {
            return new Date().getFullYear() - this.birth; // this指向window或undefined
        };
        return fn();
    }
};
//箭头函数完全修复了this的指向，this总是和函数外部的 this 指代相同的东西：
var obj = {
    birth: 1990,
    getAge: function () {
        var b = this.birth; // 1990
        var fn = () => new Date().getFullYear() - this.birth; // this 和 this.birth 指的是一个东西
        return fn();
    }
};
obj.getAge(); // 25
```
如果使用箭头函数,以前 `var that = this;` 以及 `.apply()` 和 `.call()` 这种写法就不需要了。

**箭头函数并不创建它自身执行的上下文，使得 `this` 取决于它在定义时的外部函数**。

> 箭头函数用在嵌套函数里，不要用成这样：
>
> ```javascript
> var obj = {
>     birth: 1990,
>     getAge: () => {
>         var b = this.birth; // this 是 undefined
>     }
> }
> ```
>
> 这里并没有外部函数，所以箭头函数里的 this 永远是 undefined

### generator
generator（生成器）是ES6标准引入的新的数据类型，类似于Python的generator的概念和语法。一个generator看上去像一个函数，但可以返回多次。

定义如下：
```javascript
function* foo(x) {
    yield x + 1;
    yield x + 2;
    return x + 3;
}
```
generator由`function*`定义（注意多出的`*`号），并且，除了`return`语句，还可以用`yield`返回多次。好处就是**函数只能返回一次，所以必须返回一个Array。但是，如果换成generator，就可以一次返回一个数，不断返回多次。**(其实这东西很像调试函数时候的断点！)

执行generator和调用函数不一样，调用generator对象有两个方法，一是不断地调用generator对象的`next()`方法：
```javascript
var f = fib(5);
f.next(); // {value: 0, done: false}
f.next(); // {value: 1, done: false}
f.next(); // {value: 1, done: true}
```
`next()`方法会执行generator的代码，然后，每次遇到`yield x;`就返回一个对象`{value: x, done: true/false}`，然后“暂停”。返回的`value`就是`yield`的返回值，`done`表示这个generator是否已经执行结束了。如果`done`为`true`，则`value`就是`return`的返回值。

第二个方法是直接用`for ... of`循环迭代generator对象，这种方式不需要我们自己判断`done`：
```javascript
for (var x of fib(5)) {
    console.log(x); // 依次输出0, 1, 1
}
```

因为generator可以在执行过程中多次返回，所以它看上去就像一个可以记住执行状态的函数。**看起来蛮有用的**

举个例子，要生成一个自增的ID，可以编写一个next_id()函数：
```javascript
var current_id = 0;

function next_id() {
    current_id ++;
    return current_id;
}
```

由于函数无法保存状态，故需要一个全局变量`current_id`来保存数字。现在改用generator：
```javascript
function* next_id() {
	var i = 0;
	while (true){
		yield ++i;
	}
}
```

## 标准对象
一些原则：
- 不要使用`new Number()`、`new Boolean()`、`new String()`创建包装对象；
- 用`parseInt()`或`parseFloat()`来转换任意类型到`number`；
- 用`String()`来转换任意类型到`string`，或者直接调用某个对象的`toString()`方法；
- 通常不必把任意类型转换为`boolean`再判断，因为可以直接写`if (myVar) {...}`；
- `typeof`操作符可以判断出`number`、`boolean`、`string`、`function`和`undefined`；
- 判断`Array`要使用`Array.isArray(arr)`；
- 判断`null`请使用`myVar === null`；
- 判断某个全局变量是否存在用`typeof window.myVar === 'undefined'`；

### Date
在JavaScript中，`Date`对象用来表示日期和时间。

要获取系统当前时间，用：
```javascript
var now = new Date();
now; // Wed Jun 24 2015 19:49:22 GMT+0800 (CST)
now.getFullYear(); // 2015, 年份
now.getMonth(); // 5, 月份，注意月份范围是0~11，5表示六月
now.getDate(); // 24, 表示24号
now.getDay(); // 3, 表示星期三
now.getHours(); // 19, 24小时制
now.getMinutes(); // 49, 分钟
now.getSeconds(); // 22, 秒
now.getMilliseconds(); // 875, 毫秒数
now.getTime(); // 1435146562875, 以number形式表示的时间戳
```

如果要创建一个指定日期和时间的Date对象，可以用：
```javascript
var d = new Date(2015, 5, 19, 20, 15, 30, 123);
d; // Fri Jun 19 2015 20:15:30 GMT+0800 (CST)
```

一个非常非常坑爹的地方，就是JavaScript的月份范围用整数表示是`0~11`，`0`表示一月，`1`表示二月……，所以要表示`6`月，我们传入的是`5`！

第二种创建一个指定日期和时间的方法是解析一个符合ISO 8601格式的字符串：
```javascript
var d = Date.parse('2015-06-24T19:49:22.875+08:00');
d; // 1435146562875
```
但它返回的不是`Date`对象，而是一个**时间戳**。不过有时间戳就可以很容易地把它转换为一个`Date`：
```javascript
var d = new Date(1435146562875);
d; // Wed Jun 24 2015 19:49:22 GMT+0800 (CST)
```

时间戳是个什么东西？时间戳是一个自增的整数，它表示从1970年1月1日零时整的GMT时区开始的那一刻，到现在的毫秒数。假设浏览器所在电脑的时间是准确的，那么世界上无论哪个时区的电脑，它们此刻产生的时间戳数字都是一样的，所以，时间戳可以精确地表示一个时刻，并且与时区无关。

要获取当前时间戳，可以用：
```javascript
if (Date.now) {
    alert(Date.now()); // 老版本IE没有now()方法
} else {
    alert(new Date().getTime());
}
```

### Json
JSON（JavaScript Object Notation）实际上是JavaScript的一个子集。在JSON中，一共就这么几种数据类型：
- number：和JavaScript的`number`完全一致；
- boolean：就是JavaScript的`true`或`false`；
- string：就是JavaScript的`string`；
- null：就是JavaScript的`null`；
- array：就是JavaScript的`Array`表示方式——`[]`；
- object：就是JavaScript的`{ ... }`表示方式。

#### 序列化

> 这个还是蛮有用的，可以打印出 js 中的对象

先把小明这个对象序列化成JSON格式的字符串：
```javascript
var xiaoming = {
    name: '小明',
    age: 14,
    gender: true,
    height: 1.65,
    grade: null,
    'middle-school': '\"W3C\" Middle School',
    skills: ['JavaScript', 'Java', 'Python', 'Lisp']
};

JSON.stringify(xiaoming); // '{"name":"小明","age":14,"gender":true,"height":1.65,"grade":null,"middle-school":"\"W3C\" Middle School","skills":["JavaScript","Java","Python","Lisp"]}'
```

要输出得好看一些，可以加上参数，按缩进输出：
```javascript
JSON.stringify(xiaoming, null, '  ');
```
结果：
```javascript
{
  "name": "小明",
  "age": 14,
  "gender": true,
  "height": 1.65,
  "grade": null,
  "middle-school": "\"W3C\" Middle School",
  "skills": [
    "JavaScript",
    "Java",
    "Python",
    "Lisp"
  ]
}
```
第二个参数用于控制如何筛选对象的键值，如果我们只想输出指定的属性，可以传入`Array`：
```javascript
JSON.stringify(xiaoming, ['name', 'skills'], '  ');
```
结果:
```javascript
{
  "name": "小明",
  "skills": [
    "JavaScript",
    "Java",
    "Python",
    "Lisp"
  ]
}
```
还可以传入一个函数，这样对象的每个键值对都会被函数先处理：
```javascript
function convert(key, value) {
    if (typeof value === 'string') {
        return value.toUpperCase();
    }
    return value;
}

JSON.stringify(xiaoming, convert, '  ');
```
上面的代码把所有属性值都变成大写：

```javascript
{
  "name": "小明",
  "age": 14,
  "gender": true,
  "height": 1.65,
  "grade": null,
  "middle-school": "\"W3C\" MIDDLE SCHOOL",
  "skills": [
    "JAVASCRIPT",
    "JAVA",
    "PYTHON",
    "LISP"
  ]
}
```
如果我们还想要精确控制如何序列化小明，可以给`xiaoming`定义一个`toJSON()`的方法，直接返回JSON应该序列化的数据：
```javascript
var xiaoming = {
    name: '小明',
    age: 14,
    gender: true,
    height: 1.65,
    grade: null,
    'middle-school': '\"W3C\" Middle School',
    skills: ['JavaScript', 'Java', 'Python', 'Lisp'],
    toJSON: function () {
        return { // 只输出name和age，并且改变了key：
            'Name': this.name,
            'Age': this.age
        };
    }
};

JSON.stringify(xiaoming); // '{"Name":"小明","Age":14}'
```

#### 反序列化
拿到一个JSON格式的字符串，我们直接用`JSON.parse()`把它变成一个JavaScript对象：
```javascript
JSON.parse('[1,2,3,true]'); // [1, 2, 3, true]
JSON.parse('{"name":"小明","age":14}'); // Object {name: '小明', age: 14}
JSON.parse('true'); // true
JSON.parse('123.45'); // 123.45
```
`JSON.parse()`还可以接收一个函数，用来转换解析出的属性：
```javascript
JSON.parse('{"name":"小明","age":14}', function (key, value) {
    // 把number * 2:
    if (key === 'name') {
        return value + '同学';
    }
    return value;
}); // Object {name: '小明同学', age: 14}
```

## 面向对象编程
JavaScript不区分类和实例的概念，而是通过原型（prototype）来实现面向对象编程。

prototype有点类似于继承，`A`的原型是`B`，意味着，`A`拥有`B`的全部属性。
```javascript
// 原型对象:
var Student = {
    name: 'Robot',
    height: 1.2,
    run: function () {
        console.log(this.name + ' is running...');
    }
};

function createStudent(name) {
    // 基于Student原型创建一个新对象:
    var s = Object.create(Student);
    // 初始化新对象:
    s.name = name;
    return s;
}

var xiaoming = createStudent('小明');
xiaoming.run(); // 小明 is running...
xiaoming.__proto__ === Student; // true
```

### 创建对象
JavaScript对每个创建的对象都会设置一个原型，指向它的原型对象。当我们用`obj.xxx`访问一个对象的属性时，JavaScript引擎先在当前对象上查找该属性，如果没有找到，就到其原型对象上找，如果还没有找到，就一直上溯到`Object.prototype`对象，最后，如果还没有找到，就只能返回`undefined`。

例如，创建一个`Array`对象：
```javascript
var arr = [1, 2, 3];
```
其原型链是：
```javascript
arr ----> Array.prototype ----> Object.prototype ----> null
```
`Array.prototype`定义了`indexOf()`、`shift()`等方法，因此你可以在所有的`Array`对象上直接调用这些方法。

当我们创建一个函数时：
```javascript
function foo() {
    return 0;
}
```
函数也是一个对象，它的原型链是：
```javascript
foo ----> Function.prototype ----> Object.prototype ----> null
```
由于`Function.prototype`定义了`apply()`等方法，因此，所有函数都可以调用`apply()`方法。

#### 构造函数
即用一个 `function` 来构造一个对象。相关的还需要学习 `prototype` 不过 ES6 后，都是用 class 了。

### class继承
新的关键字`class`从ES6开始正式被引入到JavaScript中。`class`的目的就是让定义类更简单。

如果用新的`class`关键字来编写`Student`，可以这样写：
```javascript
class Student {
    constructor(name) {
        this.name = name;
    }

    hello() {
        alert('Hello, ' + this.name + '!');
    }
}

var xiaoming = new Student('小明');
xiaoming.hello();
```

#### class继承
用`class`定义对象的另一个巨大的好处是继承更方便了,直接通过`extends`来实现：
```javascript
class PrimaryStudent extends Student {
    constructor(name, grade) {
        super(name); // 记得用super调用父类的构造方法!
        this.grade = grade;
    }

    myGrade() {
        alert('I am at grade ' + this.grade);
    }
}
```

通过`super(name)`来调用父类的构造函数。`PrimaryStudent`已经自动获得了父类`Student`的`hello`方法，我们又在子类中定义了新的`myGrade`方法。