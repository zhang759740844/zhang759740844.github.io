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
```

和 swift 基本一致

#### 对象

```typescript
let a: {b: string, c: string} = {b: '123', c: '234'}
```

可以像上面这样用 `{}` 包裹，直接定义一个对象类型。

### 变量声明

`let` 声明变量 `const` 声明常量。

> swift 中 `var` 声明变量，`let` 声明常量





