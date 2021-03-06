title: JS 中的正则表达式的使用
date: 2020/3/25 14:07:12  
categories: JavaScript
tags: 

 - 学习笔记

---

经常会有写脚本做文本替换的需求。一有这种需求就要去差怎么写正则表达式，非常的费时。这里把一些常用的列举一下方便以后回顾。

<!--more-->

## 正则表达式

### 元字符

|  元字符  |            说明            |
| :------: | :------------------------: |
|    $     |      匹配字符串的结束      |
|    ^     |      匹配字符串的开始      |
|    .     | 匹配除换行符以外的任意字符 |
| [a-zA-Z] |      匹配所有英文字符      |
|  [0-9]   |        匹配所有数字        |
|  [^a-z]  | 匹配除了小写字母的所有字符 |

### 重复

| 重复  |       说明       |
| :---: | :--------------: |
|   *   | 重复零次或更多次 |
|   +   | 重复一次或更多次 |
|   ?   |  重复零次或一次  |
|  {n}  |     重复n次      |
| {n,}  | 重复n次或更多次  |
| {n,m} |    重复n到m次    |

### 分支

| 分支 |   说明   |
| :--: | :------: |
| 竖线 | 相当于或 |

> 分支是指两个完整的正在表达式取或，而不是部分或
>
> |前后用（）包括：(A)|(B)
>
> 另外 | 不能有空格，否则会以为要匹配空格

### 修饰符

| 修饰符 |                           说明                           |
| :----: | :------------------------------------------------------: |
|   i    |                执行对大小写不敏感的匹配。                |
|   g    | 执行全局匹配（查找所有匹配而非在找到第一个匹配后停止）。 |
|   m    |                      执行多行匹配。                      |

### 正则表达式和通配符的不同

在 sql 的 like 语句中：`%` 任意多个， `_`表示 一个

普通的统配符： `*` 任意多个 `?` 一个

## JS 中的正则表达式

js 中使用 `正则表达式.exec(目标字符串)` 的形式处理。正则表达式以 `//` 包裹。

```js
var reg = /(.a)t/g;
var str ="1at,2at,3at";
console.log(reg.exec(str));
console.log(reg.exec(str));
console.log(reg.exec(str));
console.log(reg.exec(str));
————————————————
[ '1at', '1a', index: 0, input: '1at,2at,3at' ]
[ '2at', '2a', index: 4, input: '1at,2at,3at' ]
[ '3at', '3a', index: 8, input: '1at,2at,3at' ]
null
```

上面没有说到修饰符，但是修饰符的作用可能不明确，尤其是这个 g。g 代表的是匹配了之后再次执行不会从头开始。上面的输出中可以看到，每次执行都会找下一个匹配项。而如果不是 g，那么就会从头开始。

```js
var reg = /(.a)t/;
var str ="1at,2at,3at";
console.log(reg.exec(str));
console.log(reg.exec(str));
console.log(reg.exec(str));
console.log(reg.exec(str));
————————————————
[ '1at', '1a', index: 0, input: '1at,2at,3at' ]
[ '1at', '1a', index: 0, input: '1at,2at,3at' ]
[ '1at', '1a', index: 0, input: '1at,2at,3at' ]
[ '1at', '1a', index: 0, input: '1at,2at,3at' ]
```

> `正则表达式.exec(目标字符串)` 返回的结果一般是一个长度为1的素组，所以直接打印就行了，但是有时候会返回一个长度非1的数组，原因未知。因此最好返回结果取第一个