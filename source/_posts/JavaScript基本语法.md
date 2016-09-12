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
JS不区分整数浮点数，统一用`Number`表示：
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

#### 变量
申明一个变量用var语句，比如：
```javascript
var a; // 申明了变量a，此时a的值为undefined
var $b = 1; // 申明了变量$b，同时给$b赋值，此时$b的值为1
var s_007 = '007'; // s_007是一个字符串
var Answer = true; // Answer是一个布尔值true
var t = null; // t的值是null
```
注意只能用var申明一次。

#### strict模式
为了修补JavaScript这一严重设计缺陷，ECMA在后续规范中推出了strict模式。

启用strict模式的方法是在JavaScript代码的第一行写上：
```javascript
'use strict';
```
不支持strict模式的浏览器会把它当做一个字符串语句执行，支持strict模式的浏览器将开启strict模式运行JavaScript。

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

#### 操作字符串
字符串常见的操作如下：
```javascript
var s = 'Hello, world!';
s.length; // 13
```

要获取字符串某个指定位置的字符，使用类似Array的下标操作，索引号从0开始：
```javascript
var s = 'Hello, world!';

s[0]; // 'H'
s[6]; // ' '
s[7]; // 'w'
s[12]; // '!'
s[13]; // undefined 超出范围的索引不会报错，但一律返回undefined
```
需要特别注意的是，字符串是不可变的，如果对字符串的某个索引赋值，不会有任何错误，但是，也没有任何效果.

#### toUpperCase
`toUpperCase()`返回一个全大写的字符串

#### toLowerCase
`toLowerCase()`f返回一个全小写的字符串

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

**请注意**，直接给`Array`的`length`赋一个新的值会导致`Array`大小的变化：
```javascript
var arr = [1, 2, 3];
arr.length; // 3
arr.length = 6;
arr; // arr变为[1, 2, 3, undefined, undefined, undefined]
arr.length = 2;
arr; // arr变为[1, 2]
```

**请注意**，如果通过索引赋值时，索引超过了范围，同样会引起`Array`大小的变化：
```javascript
var arr = [1, 2, 3];
arr[5] = 'x';
arr; // arr变为[1, 2, 3, undefined, undefined, 'x']
```

**大多数其他编程语言不允许直接改变数组的大小，越界访问索引会报错。然而，JavaScript的Array却不会有任何错误。在编写代码时，不建议直接修改Array的大小，访问索引时要确保索引不会越界。**

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

#### push和pop
`push()`向`Array`的末尾添加**若干**元素，`pop()`则把`Array`的最后一个元素删除掉：
```javascript
var arr = [1, 2];
arr.push('A', 'B'); // 返回Array新的长度: 4
arr; // [1, 2, 'A', 'B']
arr.pop(); // pop()返回'B'
arr; // [1, 2, 'A']
arr.pop(); arr.pop(); arr.pop(); // 连续pop 3次
arr; // []
arr.pop(); // 空数组继续pop不会报错，而是返回undefined
arr; // []
```

#### unshift和shift
如果要往`Array`的头部添加若干元素，使用`unshift()`方法，`shift()`方法则把`Array`的第一个元素删掉：
```javascript
var arr = [1, 2];
arr.unshift('A', 'B'); // 返回Array新的长度: 4
arr; // ['A', 'B', 1, 2]
arr.shift(); // 'A'
arr; // ['B', 1, 2]
arr.shift(); arr.shift(); arr.shift(); // 连续shift 3次
arr; // []
arr.shift(); // 空数组继续shift不会报错，而是返回undefined
arr; // []
```

#### sort
`sort()`可以对当前`Array`进行排序，它会直接修改当前`Array`的元素位置，直接调用时，按照默认顺序排序：

#### reverse
reverse()把整个Array的元素给掉个个，也就是反转.

#### splice
`splice()`方法是修改`Array`的“万能方法”，它可以从指定的索引开始删除若干元素，然后再从该位置添加若干元素：
````javascript
var arr = ['Microsoft', 'Apple', 'Yahoo', 'AOL', 'Excite', 'Oracle'];
// 从索引2开始删除3个元素,然后再添加两个元素:
arr.splice(2, 3, 'Google', 'Facebook'); // 返回删除的元素 ['Yahoo', 'AOL', 'Excite']
arr; // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']
// 只删除,不添加:
arr.splice(2, 2); // ['Google', 'Facebook']
arr; // ['Microsoft', 'Apple', 'Oracle']
// 只添加,不删除:
arr.splice(2, 0, 'Google', 'Facebook'); // 返回[],因为没有删除任何元素
arr; // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']
```

#### concat
`concat()`方法把当前的`Array`和另一个`Array`连接起来，并返回一个新的`Array`：
```javascript
var arr = ['A', 'B', 'C'];
var added = arr.concat([1, 2, 3]);
added; // ['A', 'B', 'C', 1, 2, 3]
arr; // ['A', 'B', 'C']
```
请注意，`concat()`方法并没有修改当前`Array`，而是返回了一个新的`Array`。
实际上，`concat()`方法可以接收任意个元素和`Array`，并且自动把`Array`拆开，然后全部添加到新的`Array`里：
```javascript
var arr = ['A', 'B', 'C'];
arr.concat(1, 2, [3, 4]); // ['A', 'B', 'C', 1, 2, 3, 4]
```

#### join
`join()`方法是一个非常实用的方法，它把当前`Array`的每个元素都用指定的字符串连接起来，然后返回连接后的字符串：
```javascript
var arr = ['A', 'B', 'C', 1, 2, 3];
arr.join('-'); // 'A-B-C-1-2-3'
```
如果`Array`的元素不是字符串，将自动转换为字符串后再连接。

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
返回：
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


### 变量作用域
在JavaScript中，用var申明的变量实际上是有作用域的。
- 如果一个变量在函数体内部申明，则该变量的作用域为整个**函数体**(注意，var的作用域是函数体，不是语句块)，在函数体外不可引用该变量。
- 由于JavaScript的函数可以嵌套，此时，内部函数可以访问外部函数定义的变量
- 如果内部函数和外部函数的变量名重名怎么办？JavaScript的函数在查找变量时从自身函数定义开始，从“内”向“外”查找。如果内部函数定义了与外部函数重名的变量，则内部函数的变量将“屏蔽”外部函数的变量。

#### 变量提升
JavaScript的函数定义有个特点，它会先扫描整个函数体的语句，把所有申明的变量“提升”到函数顶部：
```javascript
'use strict';

function foo() {
    var x = 'Hello, ' + y;
    alert(x);
    var y = 'Bob';
}

foo();
```

虽然是`strict`模式，但语句`var x = 'Hello, ' + y;`并不报错，原因是变量`y`在稍后申明了。但是`alert`显示`Hello, undefined`，说明变量`y`的值为`undefined`。这正是因为JavaScript引擎自动提升了变量`y`的声明，但不会提升变量`y`的赋值。

#### 全局作用域
不在任何函数内定义的变量就具有全局作用域。

#### 名字空间
不同的JavaScript文件如果使用了相同的全局变量，或者定义了相同名字的顶层函数，都会造成命名冲突.减少冲突的一个方法是把自己的所有变量和函数全部绑定到一个全局变量中。例如：
```javascript
// 唯一的全局变量MYAPP:
var MYAPP = {};

// 其他变量:
MYAPP.name = 'myapp';
MYAPP.version = 1.0;

// 其他函数:
MYAPP.foo = function () {
    return 'foo';
};
```

把自己的代码全部放入唯一的名字空间`MYAPP`中，会大大减少全局变量冲突的可能。
许多著名的JavaScript库都是这么干的：jQuery，YUI，underscore等等。

#### 局部作用域
由于JavaScript的变量作用域实际上是函数内部，我们在for循环等语句块中是无法定义具有局部作用域的变量的：
```javascript
'use strict';

function foo() {
    for (var i=0; i<100; i++) {
        //
    }
    i += 100; // 仍然可以引用变量i
}
```

为了解决块级作用域，ES6引入了新的关键字`let`，用`let`替代`var`可以申明一个块级作用域的变量：
```javascript
'use strict';

function foo() {
    var sum = 0;
    for (let i=0; i<100; i++) {
        sum += i;
    }
    i += 1; // SyntaxError
}
```

#### 常量
由于var和let申明的是变量，如果要申明一个常量，在ES6之前是不行的，我们通常用全部大写的变量来表示“这是一个常量，不要修改它的值”.

ES6标准引入了新的关键字`const`来定义常量，`const`与`let`都具有块级作用域：
```javascript
'use strict';

const PI = 3.14;
PI = 3; // 某些浏览器不报错，但是无效果！
PI; // 3.14
```

注意，一定要加上`use strict`,否则会报错。

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

单独调用函数`getAge()`怎么返回了`NaN`？请注意，我们已经进入到了JavaScript的一个大坑里。
JavaScript的函数内部如果调用了`this`，那么这个`this`到底指向谁？
答案是，视情况而定！
如果以对象的方法形式调用，比如`xiaoming.age()`，该函数的`this`指向被调用的对象，也就是`xiaoming`，这是符合我们预期的。
如果单独调用函数，比如`getAge()`，此时，该函数的`this`指向**全局对象**.

更坑爹的是，如果这么写：
```javascript
var fn = xiaoming.age; // 先拿到xiaoming的age函数
fn(); // NaN
```
也是不行的！要保证`this`指向正确，必须要给出明确的上下文，必须用`obj.xxx()`的形式调用！谁调用，`this`就是指谁。上面仅仅相当于将`age`方法赋给`fn`.

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

### apply
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

另一个与`apply()`类似的方法是`call()`，唯一区别是：
- `apply()`把参数打包成`Array`再传入；
- `call()`把参数按顺序传入。 

比如调用`Math.max(3, 5, 4)`，分别用`apply()`和`call()`实现如下：
```javascript
Math.max.apply(null, [3, 5, 4]); // 5
Math.max.call(null, 3, 5, 4); // 5
```
对普通函数调用，我们通常把this绑定为null。

### 装饰器







