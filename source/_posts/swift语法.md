title: Swift 3.1 语法学习（一）
date: 2017/2/22 14:07:12  
categories: iOS
tags: 

	- Swift


------

开始啃 Swift 3.1 的官方文档，地址[Swift 官方文档](https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/Swift_Programming_Language/)。

可以参见相关的中文文档：[文档1](https://www.cnswift.org/)[文档2](http://www.swift51.com/swift3.0/) 其中有些地方翻译的不好，所以可以对照着看一看。

<!--more-->

## 基础

### 变量和常量

常量设置好就不能变，变量设置好能在之后设置不同的值。

#### 声明

用 `let` 声明常量，用 `var` 声明变量：

```swift
let maximumNumberOfLoginAttempts = 10
var currentLoginAttempt = 0
```

可以在一行里声明多个变量或常量，用逗号隔开：

```swift
var x = 0.0, y = 0.0, z = 0.0
```

（变量初始化后就不能赋给其非该类型的值了，比如上面的 x，就只能接收 double 类型的值了）

#### 类型注释

声明的时候可以注明类型：

```swift
var welcomeMessage: String
```

这样，该变量就只能接受字符串类型。可以以这样的方式一次定义多个变量：

```swift
var red, green, blue: Double
```

#### 打印

可以通过 `print(_:separator:terminator:)` 方法打印变量。其中 `separator` 表示分隔符，默认是空格，`terminator` 表示终止符，默认是回车。该方法接受多个需要打印的变量输入，例：

```swift
print("item1","item2",separator:"_",terminator:" end")
// 打印结果:"item1_item2 end"
print("item1",terminator:" end")
// 打印结果:"item1 end"
```

Swift 提供反斜杠 `\` 来达到字符串中替换变量名的效果（可以接受非字符串）：

```swift
var myName = "Zachary"
var myAge = 18
print("My name is \(myName),and my age is \(myAge)")
// 打印结果:"My name is Zachary,and my age is 18"
```

### 分号

Swift 中不需要写分号，一句一行。但是如果一句要有多个表达式，还是要用分号隔开的：

```swift
let cat = "🐱"; print(cat)
// Prints "🐱"
```

### 类型安全与类型推断

Swift 是类型安全的语言。如果代码里需要的是一个 `String`，那么你就不能传个 `Int`。不过在类型推断这个机制的作用下，你不必具体说明类型。比如：

```swift
let age1 = 18	// 默认是 Int 类型
let age2 = 18.0	// 默认是 Double 类型(Swift 默认用 Double 而不是 Float)
```

### 数字的表现方式

(省略了不同进制，科学计数等内容)

Swift 允许使用 `_` 下划线来做标识，不影响数字大小：

```swift
let oneMillion = 1_000_000
let justOverOneMillion = 1_000_000.000_000_1
```

### 数字类型转换

不能对两个**不同类型的变量**进行操作，比如：

```swift
let three = 3
let pointOneFourOneFiveNine = 0.14159
let pi = Double(three) + pointOneFourOneFiveNine	// 3.14159
let num = three + Int(pointOneFourOneFiveNine)		// 3 + 0 = 3
```

这里的 `three` 是 `Int` 型，`pointOneFourOneFiveNine` 是 `Double` 型。由于没有隐式转换，要对两个变量进行和操作必须先对其中一个进行显式类型转换。（数字3是可以直接和数字0.14159相加的，因为数字是没有类型的）

### 类型别名

类型别名为现有类型定义一个别名。定义了一个类型别名之后，你可以在任何使用原始名的地方使用别名：

```swift
typealias AudioSample = Int
var num:AudioSample = 10
```

### 布尔类型

Swift 的布尔类型叫做 Bool，两个布尔值为 true 和 false。

其它没什么特别的，有一点要说明：

```swift
// 编译不通过
let i = 1
if i {
    // this example will not compile, and will report an error
}

// 编译通过
let i = 1
if i == 1 {
    // this example will compile successfully
}
```

如果你在需要使用 Bool 类型的地方使用了非布尔值，Swift 的类型安全机制会报错。这就不像很多其他语言，非零非空就是 true。

### 元组

元组在其它的脚本语言里用的挺多了。它把多个值组合成一个复合值。**元组内的值可以是任意类型**，并不要求是相同类型。

例如一个 HTTP 状态码：

```swift
let http404Error = (404, "Not Found")
// http404Error is of type (Int, String), and equals (404, "Not Found")
```

你可以把任意顺序的类型组合成一个元组，也可以将元组分解：

```swift
let (statusCode, statusMessage) = http404Error
print("The status code is \(statusCode)")
// Prints "The status code is 404"
print("The status message is \(statusMessage)")
// Prints "The status message is Not Found"
```

这样，对应位置的变量就被赋值了。如果只需要部分元组值，可以使用 `_` 忽略，这是下划线的第二种用法，注意赋值的数量要对上：

```swift
let (justTheStatusCode, _) = http404Error
print("The status code is \(justTheStatusCode)")
// Prints "The status code is 404"
```

我们还可以直接操作元组元素的下标拿到元素：

```swift
print("The status code is \(http404Error.0)")
// Prints "The status code is 404"
print("The status message is \(http404Error.1)")
// Prints "The status message is Not Found"
```

我们也可以直接在声明的时候为元组元素命名：

```swift
let http200Status = (statusCode: 200, description: "OK")

print("The status code is \(http200Status.statusCode)")
// Prints "The status code is 200"
print("The status message is \(http200Status.description)")
// Prints "The status message is OK"
```

元组作为函数返回值的时候非常有用，适合作为临时的一组值的集合。但是不适合创建复杂的数据结构。如果要创建一个持久化的数据结构，还是建议使用类和结构体。

### 可选

可选可以用在一个值可能缺失的情况下。其实就是可能为基本类型，也可能为空。比如在强制类型转化中:

```swift
let possibleNumber = "123"
let convertedNumber = Int(possibleNumber)
// convertedNumber is inferred to be of type "Int?", or "optional Int"
```

这里面将 `possibleNumber` 强转为 Int 型，可能成功，但如果是 `"hello,world"` 就肯定失败了。所以这里 `convertedNumber` 就是一个可选类型，不一定是 Int。可选的 Int 类型用 `Int?` 表示。问号暗示包含的值是可选，也就是说可能是 Int 也可能不包含值。

#### nil

如果一个类型是可选类型，那么你可以将其值设为 `nil`:

```swift
var serverResponseCode: Int? = 404
// serverResponseCode contains an actual Int value of 404
serverResponseCode = nil
// serverResponseCode now contains no value
```

**如果变量或常量的类型不是可选的，那么就不能用 `nil` 了。当代码中的变量或者常量可能会为空的时候，总是将其设置为可选类型。**

如果定义了一个没有提供默认值的可选变量，那么默认设置为 nil；定义一个没有默认值的非可选变量，如果使用那么会报错：

```swift
var surveyAnswer: String?
print(surveyAnswer) 		// nil
var surveyAnswer: String	
print(surveyAnswer) 		// variable 'surveyAnswer' used before being initialized
```

oc 中的 nil 是一个指向不存在对象的指针。在 Swift 中 nil 不是指针，只是表示缺失值，任何类型都可以被设置为 nil（包括基本类型）。

#### 强制解析

如果你确信你的可选类型一定是有值的，那么可以在可选的变量名后面加上 `!`，表示这个可选值必然有值：

```swift
if convertedNumber != nil {
    print("convertedNumber has an integer value of \(convertedNumber!).")
}
// Prints "convertedNumber has an integer value of 123."
```

这里要注意啊，由于 `convertedNumber` 是一个可选类型，如果想要正确打印一定要加 `!`。否则就会像下面这样：

```swift
var convertedNumber = Int("123")
if convertedNumber != nil {
    print("convertedNumber has an integer value of \(convertedNumber).")
}
// Prints "convertedNumber has an integer value of Optional(123)."
```

很奇怪不是么？

####  可选绑定

可选绑定可以用在 if 和 while 语句中来对可选的值进行判断，并把值赋给一个常量或者变量，来个例子:

```swift
if let actualNumber = Int(possibleNumber) {
    print("\"\(possibleNumber)\" has an integer value of \(actualNumber)")
} else {
    print("\"\(possibleNumber)\" could not be converted to an integer")
}
// Prints ""123" has an integer value of 123"
```

这个表示，如果 `Int(possibleNumber)` 转换后有值，那么赋值给 `actualNumber`，走成功的分支，否则走失败的分支。注意，这里的 `actualNumber` 就不需要加 `!` 了，因为由于可选绑定，它已经不是一个可选类型了。

注意可选绑定的格式：

> if let constantName = someOptional {	
>
> ​	statements
>
> }

这里的 `let` 或者 `var` 是必须的，并且作用域为整个 if 判断。

一条 if 可以写多个可选绑定，用逗号隔开就行，表示 & 的关系，一个是 false 则整个判断条件是 false。

```swift
if let firstNumber = Int("4"), let secondNumber = Int("42"), firstNumber < secondNumber && secondNumber < 100 {
    print("\(firstNumber) < \(secondNumber) < 100")
}
// Prints "4 < 42 < 100"
 
if let firstNumber = Int("4") {
    if let secondNumber = Int("42") {
        if firstNumber < secondNumber && secondNumber < 100 {
            print("\(firstNumber) < \(secondNumber) < 100")
        }
    }
}
// Prints "4 < 42 < 100"
```

#### 隐式解析可选

搞了这么个可选后，如果确定有值，每次都要判断和解析可选值是非常低效的。因此就定义了一个隐式解析可选的方式:

```swift
let possibleString: String? = "An optional string."
let forcedString: String = possibleString! // requires an exclamation mark
 
let assumedString: String! = "An implicitly unwrapped optional string."
let implicitString: String = assumedString // no need for an exclamation mark
```

当可选被第一次赋值之后就可以确定之后一直有值的时候，隐式解析可选非常有用。隐式解析可选主要被用在 Swift 中类的构造过程中(暂时还不知道具体作用，后面慢慢看)

你可以把隐式解析可选当做一个可以自动解析的可选。你要做的只是声明的时候把感叹号放到类型的结尾，而不是每次取值的可选名字的结尾。如果你在隐式解析可选没有值的时候尝试取值，会触发运行时错误。和你在没有值的普通可选后面加一个惊叹号一样。如果一个变量之后可能变成 nil 的话请不要使用隐式解析可选。如果你需要在变量的生命周期中判断是否是 nil 的话，请使用普通可选类型。

## 基础运算符

### 赋值运算

和别的语言的赋值没什么区别。

对于元组的赋值，元组内元素会被立刻拆开成多个变量：

```swift
let (x, y) = (1, 2) // 现在 x 等于 1, y 等于 2 
```

补充：这里的 x 和 y 相当于被声明成立 let 类型。

另外，由于 Swift 中的 if 判断需要明确的布尔值，所以下面的代码在 Swift 中是不合法的：

```swift
if x = y {
    // This is not valid, because x = y does not return a value.
}
```

### 数值运算

和别的语言的赋值没什么区别。

加法运算符也用于字符串的拼接。注意，不是字符串要手动转换，否则不合法：

```swift
let myAge = 18
print("my age is "+myAge)			// ❎
print("my age is "+String(myAge))	// ✅
```

#### 取余

求余运算（a % b）是计算 b 的多少倍刚刚好可以容入 a，返回多出来的那部分，计算公式为：

> a = (b × 倍数) + 余数

```swift
9%4		// 1
-9%4	// -1
```

Swift 中可以对浮点数进行取余：

```swift
8 % 2.5 // 等于 0.5
```

在对负数 b 求余时，b 的符号会被忽略。这意味着 a % b 和 a % -b的结果是相同的。

#### 自增自减

自增自减已被移除

### 复合赋值

```swift
var a = 1
a += 2 // a 现在是 3
```

注意，复合赋值是没有返回值的。

### 比较运算符

和别的语言的比较没啥区别

元组也是能够比较的，从左向右比较，直到发现不同。布尔值不能被比较：

```swift
(1, "zebra") < (2, "apple")   // true because 1 is less than 2; "zebra" and "apple" are not compared
(3, "apple") < (3, "bird")    // true because 3 is equal to 3, and "apple" is less than "bird"
(4, "dog") == (4, "dog")      // true because 4 is equal to 4, and "dog" is equal to "dog"
```

### 空合并操作符

空合并操作符 `(a ?? b)`，解析一个可选的 a，如果不为空，那么返回 a!，如果为空，那么返回默认值 b。等效于如下：

```swift
a != nil ? a! : b
```

主要就是给可选量设置一个默认值，例子：

```swift
let defaultColorName = "red"
var userDefinedColorName: String?   // defaults to nil
 
var colorNameToUse = userDefinedColorName ?? defaultColorName
// userDefinedColorName is nil, so colorNameToUse is set to the default of "red"
```

### 区间操作符

#### 闭区间运算符

闭区间运算符 `a...b`，定义了一个包括 a 和 b 的所有值的区间。在 for…in 循环中非常有用：

```swift
for index in 1...5 {
    println("\(index) * 5 = \(index * 5)")
}
// 1 * 5 = 5
// 2 * 5 = 10
// 3 * 5 = 15
// 4 * 5 = 20
// 5 * 5 = 25
```

#### 半闭区间

半闭区间 `a..b`，定义一个从 a 到 b，但不包括 b 的区间。

半闭区间的实用性在于当你使用一个 0 始的列表(如数组)时，非常方便地从 0 数到列表的长度：

```swift
let names = ["Anna", "Alex", "Brian", "Jack"]
let count = names.count
for i in 0..count {
    println("第 \(i + 1) 个人叫 \(names[i])")
}
// 第 1 个人叫 Anna
// 第 2 个人叫 Alex
// 第 3 个人叫 Brian
// 第 4 个人叫 Jack
```

### 逻辑操作符

没什么区别






