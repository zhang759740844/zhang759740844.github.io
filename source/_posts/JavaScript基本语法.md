title: JavaScript基本语法
date: 2016/9/9 10:07:12  
categories: JavaScript
tags:
	- 学习笔记
---

JS 的语法，参考了廖雪峰的博客以及阮一峰的ES6

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

数组的元素可以通过索引来访问。索引的起始值为0：
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

要获取一个对象的属性，可以用一下两种方式：
```javascript
person.name; // 'Bob'
person.zipcode; // null

function getName() {
    return 'name'
}
person[getName()]
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

**字符串相当于是一个不能改变的数组。**

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

#### 展开运算符

展开运算符使用 `...` 将一个数组转为用逗号分隔的参数序列：

```javascript
console.log(1, ...[2, 3, 4], 5)
// 相当于 =>
// console.log(1, 2, 3, 4, 5)
```

#### length

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

#### 万能方法 splice

为数组添加和删除元素有很多方法，包括 push，pop，unshift，shift，slice，concat。这些都可以使用 `splice` 代替：

```javascript
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

第一个元素是起始索引(从0开始)，第二个参数是删除元素个数。后面的参数是要添加的元素。

#### join 

`join` 方法是一个非常实用的方法，用于当前`Array`的每个元素都用指定的字符串连接起来，然后返回连接后的字符串：

```javascript
var arr = ['A', 'B', 'C', 1, 2, 3];
arr.join('-'); // 'A-B-C-1-2-3'
```

#### 数组的高阶函数

##### map

`map()`方法定义在JavaScript的`Array`中，我们调用`Array`的`map()`方法，传入我们自己的函数，就得到了一个新的`Array`作为结果：

```javascript
function pow(x) {
    return x * x;
}

var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
```

`map()`将传入的函数一一作用在数组的每一个元素上。

##### reduce

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

##### filter

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

##### sort

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

### 对象

#### 基本使用

**现在 ES6 中有了 class，我觉得现在对象更像是结构体。**

定义方式如下：

```javascript
var xiaohong = {
    name: '小红',
};
```
获取分为两种形式，点语法和`[]`语法：
```javascript
xiaohong.name; // '小红'
xiaohong['name']; // '小红'
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

如果我们要检测`xiaoming`是否拥有某一属性，可以用`in`操作符，自己的和继承的都会返回 true：
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

#### 对象的属性名

##### 属性名为方法返回值

JS 中属性必须要是字符串型的，不过你也可以使用方法的返回值动态的设置属性名：

```javascript
const Birth = {
    birth: 'birth'
}

const Person = {

  name: '张三',

  //等同于birth: birth
  [Birth.birth]: '94-4-15',

  // 等同于hello: function ()...
  hello() { console.log('我的名字是', this.name); }

};
```

##### 属性的简写

ES6 中允许创建对象的时候，简写属性，直接**以变量名称作为属性名**。比如：

```javascript
let birth = '2000/01/01';

const Person = {

  name: '张三',

  //等同于birth: birth
  birth,

  // 等同于hello: function ()...
  hello() { console.log('我的名字是', this.name); }

};
```

#### Object.assign()

这个方法是非常有用的一个方法。用于**为对象添加属性，方法，克隆对象，合并对象**。第一个是目标对象，我们可以使用空对象，后面对象的属性会覆盖前面对象的属性：

```javascript
const target = {};

const source1 = { b: 2, c: 2 };
const source2 = { c: 3 };

Object.assign(target, source1, source2);
target // {b:2, c:3}
```

> 这个和闭包一样，也是指针传递，生成的 target 对象或者源 source 对象，两者任一改动都会触发另外对象的改动。
>
> 除了方法是**基本类型是值传递，对象类型是引用传递。**其他都是指针传递。
>
> （网上纠结的什么传递地址也是值传递，就当是扯淡吧，理解意思就行）

#### Object.entries()

返回一个数组，成员是参数对象自身的（不含继承的）所有可遍历属性的键值对数组：

```javascript
const obj = { foo: 'bar', baz: 42 };
Object.entries(obj)
// [ ["foo", "bar"], ["baz", 42] ]
```

配合 `for...of...` 可以拿到所有键值。

> 可以用它来将对象转化为 Map

#### JSON.stringify()

一个对象你在打印log的时候，可能会给你返回 `[object object]`。此时，你需要将对象展开，可以使用 `JSON.stringify()` 方法。

### Map和Set

JS中默认对象表达方式`{}`可以视为其他语言中的`Map`或`Dictionary`的数据结构，即一组键值对。
但是JavaScript的对象有个小问题，就是**键必须是字符串**。但实际上Number或者其他数据类型作为键也是非常合理的。
为了解决这个问题，最新的ES6规范引入了新的数据类型`Map`。

#### Map

Map 可以接受一个数组，数组元素是键值对的数组：

```javascript
var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);
m.delete('Adam'); // 删除key 'Adam'
m.get('Michael'); // 95
m.set('Zachary', 100);
```

这个结构是不是很熟悉？**`Object.entries()` 返回的就是这样的数组中包含了键值对数组的形式。**所以说，可以用 `Object.entries()` 创建 Map。因为直接把返回值作为参数传入 Map 即可。

> 对象可以变为 Map，但是 Map 不一定能变为对象。所以创建 Map 的时候的入参不能以对象的形式。

##### 方法

- `size` 属性返回 Map 结构的成员总数，类似数组的 `length`
- `set(key, value)`：设置键名`key`对应的键值为`value`，然后返回整个 Map 结构
- `get(key)`: 读取`key`对应的键值，如果找不到`key`，返回`undefined`。
- `has(key)`:返回一个布尔值，表示某个键是否在当前 Map 对象之中。
- `delete(key)`:`delete`方法删除某个键，返回`true`。如果删除失败，返回`false`。
- `clear()`:清除所有成员，没有返回值。
- `entries()`:返回所有成员的遍历器。配合 `for...of...`

####  WeakMap

WeakMap 的**键是对象类型**。如果你要往对象上添加数据，又不想干扰垃圾回收机制，就可以使用 WeakMap。

**注意弱引用的是键，而不是值**

#### Set

重复元素在Set中自动被过滤：
```javascript
var s = new Set([1, 2, 3, 3, '3']);
s; // Set {1, 2, 3, "3"}
```
注意数字`3`和字符串`'3'`是不同的元素。

##### 方法

- `add(value)`：添加某个值。
- `delete(value)`：删除某个值，返回一个布尔值
- `has(value)`：返回一个布尔值，表示该值是否为`Set`的成员。
- `clear()`：清除所有成员。

#### WeakSet

WeakSet 中**只能存对象**。并且都是弱引用。

WeakSet **不能遍历**。因为所有对象都是弱引用，随时可能消失。

### iterable
ES6标准引入了新的`iterable`类型，`Array`、`Map`和`Set`都属于`iterable`类型。`iterable` 类型其实就是

具有`iterable`类型的集合可以通过新的`for ... of`循环来遍历。用法如下：

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

### 变量的解构赋值

有两种解构赋值，分别为数组的解构赋值和对象的解构赋值

#### 数组的解构赋值

多出的部分为 undefined，可以使用展开运算符。

```javascript
let [a, b, c] = [1, 2];	// a = 1, b = 2, c = undefined
let [x, ...y] = [1, 2, 3, 4];	// x = 1, y = [2, 3, 4]
```

解构赋值允许指定默认值，当对应的值为 undefined 时，等于默认值：

```javascript
let [x, y = 'b'] = ['a']; // x='a', y='b'
```

#### 对象的解构赋值

```javascript
let { foo, bar } = { foo: "aaa", bar: "bbb" };
```

想要改变变量的名字也是可以的，上面其实等效于：

```javascript
let { foo: foo, bar: bar } = { foo: "aaa", bar: "bbb" };
```

所以如果我们要改变变量名，可以这样：

```javascript
let { foo: param1, bar: param2 } = { foo: "aaa", bar: "bbb" };
// param1 = aaa,	param2 = bbb
```

对象也是可以指定默认值的：

```javascript
var {x, y: z = 5} = {x: 1};
x // 1
z // 5
```



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

#### rest参数
由于JavaScript函数允许接收任意个参数，ES6标准引入了`rest`参数，可以将剩余的参数放在`rest`中，`rest` 是个数组（不是一定要叫 rest，任何名称都可以）：
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

> 其实相当于把 rest 展开，然后和入参对应

如果传入的参数连正常定义的参数都没填满，`rest`参数会接收一个空数组（注意不是`undefined`）。

#### 参数默认值

ES6 给 js 的函数提供了默认值：

```javascript
function Point(x = 0, y = 0) {
  this.x = x;
  this.y = y;
}

const p = new Point();
p // { x: 0, y: 0 }
```

带默认值的参数需要写在末尾。

### 特性

#### name 属性

函数的 `name`属性，返回函数的函数名：

```javascript
function foo() {}
foo.name // "foo"
```

#### 箭头函数

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

> 语法糖，类似于python中的lambda函数

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


#### 装饰器
利用`apply()`，我们可以动态改变函数的行为。

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

`apply()` 接收两个参数，第一个参数表示调用的对象，第二个参数是入参数组。此处的 `arguments` 表示的是外部 function 的入参，是 js 提供的一个参数。

#### 闭包

**这是因为 JS 中的闭包对外部变量都是指针传递，而不是引用传递。**。

iOS 中的闭包都是引用传递，外部变量相当于是以参数的形式传入的，也就是说，闭包内改变外部变量的值，外部变量不会变化。

js 中的闭包，相当于外部变量的地址都获得了。闭包内改变外部变量的值，外部变量的值相应改变。

## Promise

ES6 提供了原生的 Promise 对象。可以将异步操作以同步的形式表达出来。

### 基本用法

```javascript
const promise = new Promise(function(resolve, reject) {
  asyncFunc((success) => {
    if (success){
      resolve(value);
    } else {
      reject(error);
    }
  })
});
```

如上所示，Promise 接受一个函数，入参为成功和失败回调，函数内执行异步操作，在必要时执行成功和失败回调。

Promise 实例生成后，可以使用 `then` 方法指定 `resolve` 和 `reject` 的回调函数：

```javascript
promise.then(function(value) {
  // success
}, function(error) {
  // failure
});
```

Promise 的大致原理是通过 then 在 Promise 里注册回调，然后创建 Promise 时传入的函数会调用注册的回调函数。

**注意，创建 Promise 的时候，函数就被执行了。但是执行到 resolve 或者 reject 方法时并不立刻执行，而是等到最后**：

```javascript
new Promise((resolve, reject) => {
  resolve(1);
  console.log(2);
}).then(r => {
  console.log(r);
});
// 2
// 1
```

这是因为，Promise 将 then 注册的回调函数都包裹上了 `setTimeout（resolved，0）`，将回调函数放置到 JS 任务队列末尾。

### 链式调用`then()`

then 方法除了向 Promise 内注册成功和失败回调，还会创建并返回一个新的 Promise。这样就能形成一个链式调用。

###失败调用 `catch()` 

`catch()` 方法是`.then(null, rejection)`的别名，用于指定发生错误时的回调函数：

```javascript
getJSON('/posts.json').then(function(posts) {
  // ...
}).catch(function(error) {
  // 处理 getJSON 和 前一个回调函数运行时发生的错误
  console.log('发生错误！', error);
});
```

**注意，Promise 对象的错误具有“冒泡”性质，会一直向后传递，直到被捕获为止。也就是说，错误总是会被下一个`catch`语句捕获。**也就是说，本例中，无论是 `getJSON` 还是 `.then()` 生成的 Promise 产生的错误都会被 `catch()` 捕获。

一般来说不要在 `then()` 方法中定义失败回调。总是使用 `catch()` 方法。

`catch()` 方法返回的还是 Promise 对象，会接着运行后面的 `then()` 方法。

### 永远执行 `finally()`

```javascript
promise
.then(result => {···})
.catch(error => {···})
.finally(() => {···});
```

上面代码中，不管`promise`最后的状态，在执行完`then`或`catch`指定的回调函数以后，都会执行`finally`方法指定的回调函数。

### 合并多个Promise `all()`

`Promise.all`方法接受一个 Promise 数组，用于将多个 Promise 实例，包装成一个新的 Promise 实例：

```javascript
const p = Promise.all([p1, p2, p3]);
p.then(function (posts) {
  // ...
}).catch(function(reason){
  // ...
});
```

`p`的状态由`p1`、`p2`、`p3`决定，分成两种情况。

（1）只有`p1`、`p2`、`p3`的状态都变成`fulfilled`，`p`的状态才会变成`fulfilled`，此时`p1`、`p2`、`p3`的**返回值组成一个数组，传递给`p`的回调函数**。

（2）只要`p1`、`p2`、`p3`之中有一个被`rejected`，`p`的状态就变成`rejected`，此时**第一个被`reject`的实例的返回值，会传递给`p`的回调函数**。

### 合并多个Promise `race()`

`Promise.race`方法同样是将多个 Promise 实例，包装成一个新的 Promise 实例：

```javascript
const p = Promise.race([p1, p2, p3]);
```

上面代码中，只要`p1`、`p2`、`p3`之中有一个实例率先改变状态，`p`的状态就跟着改变。那个率先改变的 Promise 实例的返回值，就传递给`p`的回调函数。

## Proxy

Proxy 就如字面所说的，是 ES6 提供的代理模式的封装。ES6 原生提供 Proxy 构造函数，用来生成 Proxy 实例:

```javascript
var proxy = new Proxy(target, handler);
```

第一个参数是被代理对象，第二个参数是要拦截的行为对象。当调用代理对象的被拦截行为时，触发代理方法。常用的拦截行为有以下几种：

### get(),set()

```javascript
var obj = new Proxy({}, {
  get: function (target, key, receiver) {
    console.log(`getting ${key}!`);
    return Reflect.get(target, key, receiver);
  },
  set: function (target, key, value, receiver) {
    console.log(`setting ${key}!`);
    return Reflect.set(target, key, value, receiver);
  }
});

obj.count = 1
//  setting count!
++obj.count
//  getting count!
//  setting count!
//  2
```

其中，代理对象是 `obj`，被代理的对象是 `{}` 空对象，拦截的行为是 get，set 方法。也就是说，任何对于 `obj` 的读取写入操作都将执行代理方法。

get，set 方法的参数也很好理解。`target` 表示被代理对象，`key` 表示要进行读写操作的属性，`value` 就是写操作的值。`receiver` 就是代理对象 `obj`。（先不用关 `Reflect`）。

如果`handler`没有设置任何拦截，那就等同于直接通向原对象：

```javascript
var target = {};
var handler = {};
var proxy = new Proxy(target, handler);
proxy.a = 'b';
target.a // "b"
```

### apply()

`apply` 方法**拦截函数的调用、`call `和 `apply` 操作**。可以**接受三个参数：目标对象(函数）、目标对象上下文对象(this)、目标对象的参数数组**。

```javascript
var twice = {
  apply (target, context, args) {
    return Reflect.apply(...arguments) * 2;
  }
};
function sum (left, right) {
  return left + right;
};
var proxy = new Proxy(sum, twice);
proxy(1, 2) // 6
proxy.call(null, 5, 6) // 22
proxy.apply(null, [7, 8]) // 30
```

注意这里的 `...arguments` 表示的是 `target`,`context`以及`args`的全部。

### construct()

`construct` 方法用于**拦截 `new` 命令**。接受两个参数：**目标对象(类对象)，构建函数的参数对象**。

```javascript
class A {
    constructor() {
        this.a = 1
    }
}

let Aproxy = new Proxy(A, {
    construct: function(target, args){
        return new target(...args)
    }
});

let a = new Aproxy()
console.log(a.a) //1
```

注意，**拦截的是 `new` 命令，不是拦截的 `constructor` 命令**，所以，拦截方法要返回一个 `target` 创建的新对象。

## class
### 基本用法

#### 创建和使用

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

#### name 属性

上面说到，函数有 name 属性，可以返回函数的名字。ES6 的 class 其实就是 ES5 函数的包装。所以 Class 也包含 name 属性：

```javascript
class Point {}
Point.name // "Point"
```

#### get set 方法

在“类”的内部可以使用`get`和`set`关键字，对某个属性设置存值函数和取值函数，拦截该属性的存取行为：

```javascript
class MyClass {
  constructor() {
    // ...
  }
  get prop() {
    return 'getter';
  }
  set prop(value) {
    console.log('setter: '+value);
  }
}

let inst = new MyClass();

inst.prop = 123;
// setter: 123

inst.prop
// 'getter'
```

#### static 方法

如果在一个方法前，加上`static`关键字，表示类方法:

```javascript
class Foo {
  static classMethod() {
    return 'hello';
  }
}

Foo.classMethod() // 'hello'

var foo = new Foo();
foo.classMethod()
// TypeError: foo.classMethod is not a function
```

> ES6 明确规定，Class 内部只有静态方法，没有静态属性。

#### new.target 属性

ES6 为`new`命令引入了一个`new.target`属性，该属性一般用在构造函数之中，返回`new`命令作用于的那个构造函数。需要注意的是，子类继承父类时，`new.target`会返回子类。利用这个特点，可以写出不能独立使用、必须继承后才能使用的**接口**：

```javascript
class Shape {
  constructor() {
    if (new.target === Shape) {
      throw new Error('本类不能实例化');
    }
  }
}

class Rectangle extends Shape {
  constructor(length, width) {
    super();
    // ...
  }
}

var x = new Shape();  // 报错
var y = new Rectangle(3, 4);  // 正确
```



### class继承

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

## export 与 import

### 基本用法

```javascript
export let name = 'zachary'
export function getName() {
    return 'zachary'
}

等价于=> export {name, getName}
```

注意，不能直接写`export name`

```javascript
import {name, getName} from './myName'
```

其实相当于对象的解构赋值。

### 重命名 as

导出导入的名字有时候可能需要改变，所以提供了 as 这个关键字：

```javascript
export {name as yourName, getName as getYourName}
import {yourName as myName, getYourName as getMyName} from './myName'
```

导出的时候用 `yourName` 作为内部 `name` 的别名。导入的时候用 `myName` 作为 `youName` 的别名。

### default

有时候，我们不需要知道导出的叫什么，可以使用 default ：

```javascript
export default name
import myName from './myName'

相当于 =>
export {name as default}
import {default as myName} from './myName'
```

其实就是 as 的语法糖。注意，import 的时候就不需要大括号了。

### 与 CommonJS 中 require()，module.exports 的不同

`module.exports = {}` 直接导出一个对象。`let a = require('../module.js')`直接从路径获取导出的对象。

> 这种方式导出的是一个整的对象，如果要导出多个，需要作为对象的各个属性。而 ES6 的导出相当于直接将导出对象解构赋值了。

## 归类

### … 的使用场景

`…` 一般用在数组前，**把数组转为用逗号隔开的一个个值**。可以正用逆用都是等价的：

```javascript
...[a, b, c] => a, b, c
a, b, c => ...[a, b, c]
```

一般用于：

1. 将数组拆开作为入参
2. 将入参合并为数组
3. 数组解构赋值

```javascript
// 1. 做入参
someFunc(1, ...[2, 3, 4])    => someFunc(1, 2, 3, 4)
// 2. 合并入参
function someFunc(1, ...args) {}	=> args = [2, 3, 4]
// 3. 数组解构 类似于合并入参
let [1, ...args] = [1, 2, 3, 4]		=> args = [2, 3, 4]
```

另外，在 JSX 中，`...` 可以把对象中的键值对拿出来，以等号连接。

### 对象和 Map

对象和 Map 初步的不同就在于键是否是字符串。

#### 创建与增减键值对

对象：

```javascript
let obj = {}
obj.haha = '123'
delete obj.haha
```

Map：

```javascript
let map = new Map()
map.set('haha','123')
map.get('haha')
map.delete('haha')
```

#### 遍历

对象：

```javascript
Object.entries(obj) => [['haha', '123'], ['lala', '234']]
```

Map：

```javascript
map.entries() => [['haha', '123'], ['lala', '234']]
```



## 没有看的

- symbol 部分
- reflect 部分(感觉可以直接调用，不需要使用 reflect)
- generator 部分(习惯于用promise的回调写法，不习惯这种写法)
- async 部分(同 generator)
- ​