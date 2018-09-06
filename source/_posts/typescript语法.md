title: TypeScript语法
date: 2018/9/5 14:07:12  
categories: JavaScript
tags: 

	- 学习笔记
	

---

学习一下 typescript 2.7 的语法

<!--more-->

### 基础类型

声明类型的方式和 swift 一致，**声明类型后不能更改，同时允许隐式赋值**。

#### 布尔

```typescript
let isDone: boolean = false
```

#### 数字

```typescript
let decLiteral: number = 6
```

#### 字符串

```typescript
let name: string = 'bob'
```

#### 数组

```typescript
let list: number[] = [1, 2, 3]
let list: Array<number> = [1, 2, 3]
```

> 和 swift 略有不同，swift 将类型放在括号内
>
> 类似 `let list: [int] = [1, 2, 3]`

#### 元组

```typescript
let x: [string, number] = ['hello', 10]
```

> 和 swift 也略有不同，swift 是圆括号
>
> 类似 `let x: (string, int) = ('hello', 10)`

#### 枚举

```typescript
enum Color {
  Red = 'red',
  Yellow = 'yellow'
}
let c: Color = Color.Red

enum Num {
  one = 1,
  two = 2,
}
let c: Num = Num.one
let d: String = Num[1]
```

枚举其实就是生成一个对象，上面的 `Color` `Num` 打印出来：

```typescript
{ Red: 'red', Yellow: 'yellow' }
{ '1': 'one', '2': 'two', one: 1, two: 2 }
```

**注意，使用数字作为枚举值，生成的对象枚举值也会成为对象的键。**这样不仅可以通过枚举对象拿到枚举值，还可以通过枚举值拿到枚举对象的字符串。

#### any

`any` 表示任意类型。**与 `Object` 不同，`Object`类型的变量只是允许你给它赋任意值 - 但是却不能够在它上面调用任意的方法，`any` 则可以直接调用，不管有没有这个方法。**

```typescript
let notSure: any = 4
notSure.ifItExists()
```

#### void

`void` 表示没有类型，一般用在函数返回：

```typescript
function warnUser(): void {
    alert("This is my warning message");
}
```

#### 类型断言

```typescript
let someValue: any = "this is a string"
let strLength: number = (someValue as string).length
let strLength: number = (<string>someValue).length
```

第一种方式和 swift 基本一致。第二种方式的强转一般是圆括号，这里是尖括号。

#### 对象

```typescript
let a: {b: string, c: string} = {b: '123', c: '234'}
```

可以像上面这样用 `{}` 包裹，直接定义一个对象类型。

### 变量声明

`let` 声明变量 `const` 声明常量。

> swift 中 `var` 声明变量，`let` 声明常量

### 接口

**ts 中的接口并不像在其它语言里一样，传入的对象要实现这个接口，ts中只要传入的对象满足接口的要求，那么它就是被允许的。**

```typescript
interface LabelledValue {
  label: string;
}

function printLabel(labelledObj: LabelledValue) {
  console.log(labelledObj.label);
}

let myObj = {size: 10, label: "Size 10 Object"};
printLabel(myObj);
```

上面的例子中，只要入参满足存在 `label` 属性即可。

#### 可选属性

有些属性不是必须的，可选属性名字定义的后面加一个`?`符号：

```typescript
interface SquareConfig {
  color?: string;
  width?: number;
}

function createSquare(config: SquareConfig): {color: string; area: number} {
  let newSquare = {color: "white", area: 100};
  if (config.color) {
    newSquare.color = config.color;
  }
  if (config.width) {
    newSquare.area = config.width * config.width;
  }
  return newSquare;
}

let mySquare = createSquare({color: "black"});
```

#### 只读属性

```typescript
interface Point {
    readonly x: number;
    readonly y: number;
}
```

通过赋值一个对象字面量来构造一个`Point`。 赋值后， `x`和`y`再也不能被改变了。

`ReadonlyArray<T>`类型，它与`Array<T>`相似，只是把所有可变方法去掉了，因此可以确保数组创建后再也不能被修改：

```typescript
let a: number[] = [1, 2, 3, 4];
let ro: ReadonlyArray<number> = a;
ro[0] = 12; // error!
ro.push(5); // error!
ro.length = 100; // error!
a = ro; // error!
```

#### 函数类型

```typescript
interface SearchFunc {
  (source: string, subString: string): boolean;
}

let mySearch: SearchFunc;
mySearch = function(src, sub) {
    let result = src.search(sub);
    return result > -1;
}
```

函数的参数名不需要与接口里定义的名字相匹配。如果你不想指定类型，TypeScript的类型系统会推断出参数类型

#### 类类型

和 Java 一样，TS 用它强制一个类去符合某种契约，实现接口使用关键字 `implements`：

```typescript
interface ClockInterface {
    currentTime: Date
    setTime(d: Date)
}

class Clock implements ClockInterface {
    currentTime: Date;
    setTime(d: Date) {
        this.currentTime = d
    }
    constructor(h: number, m: number) { }
}
```

#### 继承接口

接口继承使用关键字 `extends`：

```typescript
interface Shape {
    color: string;
}

interface PenStroke {
    penWidth: number;
}

interface Square extends Shape, PenStroke {
    sideLength: number;
}

let square = <Square>{};
square.color = "blue";
square.sideLength = 10;
square.penWidth = 5.0;
```

### 类

#### 继承

继承和 Java 非常类似。子类的构造函数中需要调用 `super` 父类的构造函数。

#### public

TS 中默认成员是 public 的。

