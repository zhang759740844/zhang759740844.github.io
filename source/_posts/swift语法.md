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

> 变量常量都可以先声明，不用立即赋值。
>
> 但是在获取的时候，只有可选变量会被默认设置为 nil。
>
> 其余非可选变量常量，以及可选常量，在没有赋初值的情况下读取，都会产生异常。
>
> (可选的常量如果默认为 nil，那就没有意义了，所以强制可选常量也抛出异常)

#### 类型注释

声明的时候可以注明类型：

```swift
var welcomeMessage: String
```

这样，该变量就只能接受字符串类型。可以以这样的方式一次定义多个变量：

```swift
var red, green, blue: Double
```

> 如果没有值，就一定要标注类型，如果有值，就可以通过类型推断。
>
> 反正一定要在声明的时候确定类型

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

> 这里反斜杠类似于转义，所以还是要在字符串里出现的，不能是直接 print(\\(myName))

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

> 这里类型推断是指没有在后面写上类型的，比如  `let age1: Int` 就已经表示是非空的 Int 型了，就不可能再推断了

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

> 感觉没啥用，还容易让人误解

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

> if 条件语句必须是一个有值的表达式
>
> 条件判断不用括号，因为不会产生二意

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

1. 这里面将 `possibleNumber` 强转为 Int 型，可能成功，但如果是 `"hello,world"` 就肯定失败了。所以这里 `convertedNumber` 就是一个可选类型，不一定是 Int。可选的 Int 类型用 `Int?` 表示。问号暗示包含的值是可选，也就是说可能是 Int 也可能不包含值。
2. 上面 possibleNumber 由于类型推断，一定是不可选类型，除非将 `possibleNumber` 写成 `let possibleNumber: String? = "123"`，才能表示可选。
3. `convertedNumber` 是一个可选类型，**在对可选类型进行赋值以外的操作的时候，需要使用 `!` 或者 `?` 包裹**。

元组也可以被设置为可选，在元组后添加 `?`：

```swift
let a: (Int,Int)? = nil
```

还有一种需要区别的是元组内元素的可选，在可选的类型后添加 `?`：

```swift
let a: (String?,Int) = (nil,2)  // √
let a = (nil,2) 				// × 没有声明，那就表示是非可选
```

> 直接赋值的都能通过赋给的值进行类型推断，而不是可选类型；前面直接 `var str: String` 声明但没有赋值的也是**非可选**的。
>
> **可选变量的声明，一定要手动标明**，即一定要加上 ?,即 `let possibleNumber: String?`，否则就是不可选的

#### nil

如果一个类型是可选类型，那么你可以将其值设为 `nil`:

```swift
var serverResponseCode: Int? = 404
// serverResponseCode contains an actual Int value of 404
serverResponseCode = nil
// serverResponseCode now contains no value
```

如果变量或常量的类型不是可选的，那么就不能用 `nil` 了。**反之亦然，如果代码中的变量或者常量可能会为空的时候，总是将其设置为可选类型。**

如果定义了一个没有提供默认值的可选变量，那么默认设置为 nil；定义一个没有默认值的非可选变量，如果使用那么会报错：

```swift
var surveyAnswer: String?
print(surveyAnswer) 		// nil
var surveyAnswer: String	
print(surveyAnswer) 		// variable 'surveyAnswer' used before being initialized
```

> 上面是 var 类型，let 类型的常量，无论是否可选，使用前必须初始化。
>
> 可能是因为如果没有初始化，那就要被推断为 nil，且不能改变。这种不是由程序上下文设置的 nil，而是推断出的 nil 没有意义，所以索性抛出个异常。

oc 中的 nil 是一个指向不存在对象的指针。在 Swift 中 nil 不是指针，只是表示缺失值，任何类型都可以被设置为 nil（包括基本类型）。



#### 强制解析

 如果你确信你的**可选类型**一定是有值的，那么可以在可选的变量名后面加上 `!`，表示这个可选值必然有值：

```swift
if convertedNumber != nil {
    print("convertedNumber has an integer value of \(convertedNumber!).")
}
// Prints "convertedNumber has an integer value of 123."
```

如果想要对一个可选类型进行赋值以外的操作，必须使用强制解析，因为只能对非空对象进行操作：

```swift
print(convertedNumber! - 12)	//√
print(convertedNumber - 12)		//×
print(convertedNumber? - 12)	//×
```



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

> 这其实就是判断非空操作的语法糖

注意可选绑定的格式：

> if let constantName = someOptional {	
>
> ​	statements
>
> }

**这里的 `let` 或者 `var` 是必须的，并且作用域为整个 if 判断如果 `Int(possibleNumber)` 存在，就声明了 `actualNumber`，如果不存在就相当于没有声明这个变量。**

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

可以都用逗号，但是有可选绑定就不能用 `&&`

> 可选绑定的生命周期在函数内，出了函数，这个变量就回收了。

#### 隐式解析可选

搞了这么个可选后，如果确定有值，每次都要判断和解析可选值是非常低效的。因此就定义了一个隐式解析可选的方式:

```swift
let possibleString: String? = "An optional string."
// 不能直接 let forcedString：String = possibleString 因为，一个是可选类型，一个是非可选类型
let forcedString: String = possibleString! // requires an exclamation mark
 
let assumedString: String! = "An implicitly unwrapped optional string."
let implicitString: String = assumedString // no need for an exclamation mark
```

当可选被第一次赋值之后就可以确定之后一直有值的时候，隐式解析可选非常有用。隐式解析可选主要被用在 Swift 中类的构造过程中(参考无主解析)

**一个隐式解析可选类型其实就是一个普通的可选类型，但是可以被当做非可选类型来使用，**并不需要每次都使用解析来获取可选值。你可以把隐式解析可选当做一个可以自动解析的可选。你要做的只是声明的时候把感叹号放到类型的结尾，而不是每次取值的可选名字的结尾。**其实就是本质上是可选，但是表面上(指的是写法上)当做非可选用。**



> **如果你在隐式解析可选没有值的时候尝试取值，会触发运行时错误**。和你在没有值的普通可选后面加一个惊叹号一样。

> 如果一个变量之后可能变成 nil 的话请不要使用隐式解析可选。如果你需要在变量的生命周期中判断是否是 nil 的话，请使用普通可选类型。

## 基础运算符

### 赋值运算

和别的语言的赋值没什么区别。

对于元组的赋值，元组内元素会被立刻拆开成多个变量：

```swift
let (x, y) = (1, 2) // 现在 x 等于 1, y 等于 2 
```

补充：这里的 x 和 y 相当于被声明成为 let 类型。

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

> 由于自增自减在 for 循环中使用的多，并且 swift 中现在 for 循环已经不是 c 那种 for 循环了。所以就被干掉了
>
> 可以使用复合赋值替代

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

### 三元选择操作符

和其他语言的三元操作符没有任何区别

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

半闭区间 `a..<b`，定义一个从 a 到 b，但不包括 b 的区间。

半闭区间的实用性在于当你使用一个 0 始的列表(如数组)时，非常方便地从 0 数到列表的长度：

```swift
let names = ["Anna", "Alex", "Brian", "Jack"]
let count = names.count
for i in 0..<count {
    print("第 \(i + 1) 个人叫 \(names[i])")
}
// 第 1 个人叫 Anna
// 第 2 个人叫 Alex
// 第 3 个人叫 Brian
// 第 4 个人叫 Jack
```

>注意现在半闭区间必须是 `..<`，不是之前的 `..`

### 逻辑操作符

没什么区别

## 字符串与字符

### 初始化字符串

两种初始化字符串的方式：

```swift
var emptyString = ""               // empty string literal
var anotherEmptyString = String()  // initializer syntax
// these two strings are both empty, and are equivalent to each other
```

判断是否为空，用 `isEmpty` 方法：

```swift
if emptyString.isEmpty {
    print("Nothing to see here")
}
// Prints "Nothing to see here"
```

### 字符串可变性

oc 中通过 `NSString` 和 `NSMutableString` 区别是否可以修改字符串。Swift 中只通过是常量还是变量来判断：

```swift
var variableString = "Horse"
variableString += " and carriage"
// variableString is now "Horse and carriage"
 
let constantString = "Highlander"
constantString += " and another Highlander"
// this reports a compile-time error - a constant string cannot be modified
```

### 字符串是值传递

Swift 中，如果创建了一个新的字符串，那么当其进行常量、变量赋值操作或在函数/方法中传递时，都会对已有字符串值创建新副本，并对该新副本进行传递或赋值。

> let 相当于 oc 中 const 的 NSString
>
> var 相当于 oc 中可变的 NSString
>
> oc 中的 NSMutableString 是引用传递，swift 中没有和其对应的



Swift 默认字符串拷贝的方式保证了在函数/方法中传递的是字符串的值，其明确了无论该值来自于哪里，都是您独自拥有的。您可以放心您传递的字符串本身不会被更改。

### 使用字符

可以通过 `String` 的 `characters` 属性获取字符串的字符：

```swift
for character in "Dog!🐶".characters {
    print(character)
}
// D
// o
// g
// !
// 🐶
```

> swift 中的字符串没有了 length 这个属性。要获取长度还得通过 characters 这个数组属性。拿到这个数组的 count 就是字符串中的长度。

可以直接创建一个字符：

```swift
let exclamationMark: Character = "!"
```

也可以创建一个字符数组，再转换为字符串：

```swift
let catCharacters: [Character] = ["C", "a", "t", "!", "🐱"]
let catString = String(catCharacters)
print(catString)
// Prints "Cat!🐱"
```

### 连接字符串

这个其实在前面说过了，就是能通过 `+` 或者 `+=` 把两个字符串连起来：

```swift
let string1 = "hello"
let string2 = " there"
var welcome = string1 + string2
// welcome now equals "hello there"

var instruction = "look over"
instruction += string2
// instruction now equals "look over there"
```

需要注意，不能直接连接字符串和字符，需要通过 `append` 方法：

```swift
let exclamationMark: Character = "!"
welcome.append(exclamationMark)
// welcome now equals "hello there!"
```

### 字符串插入

这个之前也用过了，就是在字符串中添加一个占位符：

```swift
let multiplier = 3
let message = "\(multiplier) times 2.5 is \(Double(multiplier) * 2.5)"
// message is "3 times 2.5 is 7.5"
```

括号内不能包含未转义的 `\`，回车或者换行。

## 集合类型

Swift 提供 array，sets，dictionaries 来存储数据。在同一个集合中的元素的类型必须是相同的。

### 可变集合

和 String 类似，如果你把一个集合赋给一个 `var` 类型的变量，那么你可以动态地对集合操作；**如果你把一个集合赋给一个 `let` 的常量，那么这个集合就是不可变的**（包括数组的长度，以及数组内的对象的地址，但是可以改变对象的属性）。

### 数组

#### 数组的简写语法

Swift 数组的类型被写作 `Array<Element>`，另一种简写方式是 `[Element]`。更推荐用后一种方式：

```swift
var nums1:Array<Int> = [12,34,12]
var nums2:[Int] = [12,12,32]
```



#### 创建一个空数组

可以用以下初始化方式来创建一个确定类型的空数组：

```swift
var someInts = [Int]()
print("someInts is of type [Int] with \(someInts.count) items.")
// Prints "someInts is of type [Int] with 0 items."
```

要注意变量 `someInts` 会被推断为 `[Int]` 类型。

上面的操作主要还是为了明确数组的类型。**一个不明确的类型的数组声明，比如 `var someInts = []`是不合法的。但是如果数组本身的类型已经明确了，就可以直接用 `[]` 置空了**：

```swift
var someInts = [Int]()

someInts.append(3)
// someInts now contains 1 value of type Int
someInts = []
// someInts is now an empty array, but is still of type [Int]
```

#### 创建一个有默认值的数组

反正就是这个语法，用处大不大我就不知道了

```swift
var threeDoubles = Array(repeating: 0.0, count: 3)
// threeDoubles is of type [Double], and equals [0.0, 0.0, 0.0]
```

> 没啥用

#### 连接两个数组

如果两个数组类型相同，那么可以用 `+` 连接两个数组，返回一个新数组：

```swift
var anotherThreeDoubles = Array(repeating: 2.5, count: 3)
// anotherThreeDoubles is of type [Double], and equals [2.5, 2.5, 2.5]
 
var sixDoubles = threeDoubles + anotherThreeDoubles
// sixDoubles is inferred as [Double], and equals [0.0, 0.0, 0.0, 2.5, 2.5, 2.5]
```

#### 创建一个数组

下面例子是创建一个保存字符串的数组：

```swift
var shoppingList: [String] = ["Eggs", "Milk"]
// shoppingList has been initialized with two initial items
```

由于类型推断，我们可以省略 `[String]`：

```swift
var shoppingList = ["Eggs", "Milk"]
```

> **array 和 String 一样，也是值引用。**
>
>  `var array1 = array2` 在堆中开辟了两块内存空间，而不是指向同一个地址。可以分别处理 array1 和 array2

#### 存取和更改数组

通过 `count` 属性获得数组长度：

```swift
print("The shopping list contains \(shoppingList.count) items.")
// Prints "The shopping list contains 2 items."
```

通过 `isEmpty` 属性获得数组是否为空：

```swift
if shoppingList.isEmpty {
    print("The shopping list is empty.")
} else {
    print("The shopping list is not empty.")
}
// Prints "The shopping list is not empty."
```

通过 `append(_:)` 方法添加新项：

```swift
shoppingList.append("Flour")
// shoppingList now contains 3 items, and someone is making pancakes
```

通过 `+=` 添加新项：

```swift
shoppingList += ["Baking Powder"]
// shoppingList now contains 4 items
shoppingList += ["Chocolate Spread", "Cheese", "Butter"]
// shoppingList now contains 7 items
```

通过下标获取数组元素:

```swift
var firstItem = shoppingList[0]
// firstItem is equal to "Eggs"
```

同样的方法设置数组元素（数组下标不能越界）：

```swift
shoppingList[0] = "Six eggs"
// the first item in the list is now equal to "Six eggs" rather than "Eggs"
shoppingList[4...6] = ["Bananas", "Apples"]
// shoppingList now contains 6 items
```

> 这里 4…6 是闭区间运算符，表示三个值。这里只赋了两个值 `["Bananas", "Apples"]`，所以4，5被修改，6 被 remove 掉。如果赋四个值 `["Bananas", "Apples","Pear","Cherry"]` 呢？4，5，6被修改，最后一个插入数组中变为 7，原来数组中的 7 及以后顺延为 8 及以后。

通过 `insert(_:at:)` 在指定位置插入：

```swift
shoppingList.insert("Maple Syrup", at: 0)
// shoppingList now contains 7 items
// "Maple Syrup" is now the first item in the list
```

通过 `remove(at:)` 删除指定位置数组内容，并返回**删除的那个内容**（如果你用不到返回值，可以忽略）：

```swift
let mapleSyrup = shoppingList.remove(at: 0)
// the item that was at index 0 has just been removed
// shoppingList now contains 6 items, and no Maple Syrup
// the mapleSyrup constant is now equal to the removed "Maple Syrup" string
```

通过 `removeLast()` 删除最后一个元素：

```swift
let apples = shoppingList.removeLast()
// the last item in the array has just been removed
// shoppingList now contains 5 items, and no apples
// the apples constant is now equal to the removed "Apples" string
```

#### 迭代数组

通过 `for-in` 循环取出数组元素：

```swift
for item in shoppingList {
    print(item)
}
// Six eggs
// Milk
// Flour
// Baking Powder
// Bananas
```

如果还需要数组元素对应的下标，可以使用 `enumerated()` 方法。该方法可以返回数组元素和数组元素下标所组成的元组：

```swift
for (index, value) in shoppingList.enumerated() {
    print("Item \(index + 1): \(value)")
}
// Item 1: Six eggs
// Item 2: Milk
// Item 3: Flour
// Item 4: Baking Powder
// Item 5: Bananas
```

### 字典

和数组类似。字典的键和值的类型都应该是分别相同的。键必须不重复，且是可哈希的（Swift 的所有基本类型都是可哈希的）

#### 字典的简写语法

Swift 数组的类型被写作 `Dictionary<Key,Value>`，另一种简写方式是 `[Key:Value]`。更推荐用后一种方式：

```swift
var nums1: Dictionary<Int,String> = [12:"123",34:"1234",13:"131"]
var nums2: [Int:String] = [12:"123",34:"1234",13:"131"]
```

注意一个是逗号，一个是冒号。

#### 创建一个空字典

可以通过下面方法初始化一个空字典：

```swift
var namesOfIntegers = [Int: String]()
// namesOfIntegers is an empty [Int: String] dictionary
```

初始化后，变量的类型就确定了。即使清空字典，字典类型也不会变:

```swift
namesOfIntegers[16] = "sixteen"
// namesOfIntegers now contains 1 key-value pair
namesOfIntegers = [:]
// namesOfIntegers is once again an empty dictionary of type [Int: String]
```

#### 创建一个字典

通过下面方式创建一个字典，同样由于类型推断，不写明类型也是可以的：

```swift
var airports: [String: String] = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
var airports = ["YYZ": "Toronto Pearson", "DUB": "Dublin"]
```

#### 存取和更改数组

其他没什么区别，多了一个 `updateValue(_:forKey:)` 方法，在对特定键**设置或更新**值时，也可以使用该方法来替代下标。但与下标不同的是，该方法在更新一个值之后，会返回原来的老值。

该方法返回一个可选类型的值。如果字典里存的是 String 类型，那么该方法返回的就是 `String?`。这个可选值可以用来判断，这个方法的操作是更新还是设置。如果原来存在就非 nil，如果原来不存在就是 nil： 

```swift
if let oldValue = airports.updateValue("Dublin Airport", forKey: "DUB") {
    print("The old value for DUB was \(oldValue).")
}
// Prints "The old value for DUB was Dublin."
```

当然，还可以直接用下标的方式来获取值，由于可能为空，所以返回的也是可选类型：

```swift
if let airportName = airports["DUB"] {
    print("The name of the airport is \(airportName).")
} else {
    print("That airport is not in the airports dictionary.")
}
// Prints "The name of the airport is Dublin Airport."
```

可以直接操作下标，将键对应的值设置为 nil，来删除相应键值对：

```swift
airports["APL"] = "Apple International"
// "Apple International" is not the real airport for APL, so delete it
airports["APL"] = nil
// APL has now been removed from the dictionary
```

删除的话还可以通过专门的方法 `removeValue(forKey:)` 方法实现，该方法移除指定键的值后，返回移除了的值，如果本生就没有需要移除的键就返回 nil：

```swift
if let removedValue = airports.removeValue(forKey: "DUB") {
    print("The removed airport's name is \(removedValue).")
} else {
    print("The airports dictionary does not contain a value for DUB.")
}
// Prints "The removed airport's name is Dublin Airport."
```

#### 迭代字典

通过 `for-in` 循环获得键值对。键值对用元组存储：

```swift
for (airportCode, airportName) in airports {
    print("\(airportCode): \(airportName)")
}
// YYZ: Toronto Pearson
// LHR: London Heathrow
```

也可以通过 `keys` 和 `values` 属性，分别拿到对应集合：

```swift
for airportCode in airports.keys {
    print("Airport code: \(airportCode)")
}
// Airport code: YYZ
// Airport code: LHR
 
for airportName in airports.values {
    print("Airport name: \(airportName)")
}
// Airport name: Toronto Pearson
// Airport name: London Heathrow
```

如果想要得到获得字典的键或者值的数组，可以通过 `keys` 和 `values` 属性进行初始化：

```swift
let airportCodes = [String](airports.keys)
// airportCodes is ["YYZ", "LHR"]
 
let airportNames = [String](airports.values)
// airportNames is ["Toronto Pearson", "London Heathrow"]
```

上面见过创建空数组的方式是 `[String]()`，我们也可以通过这种方式创建带值的数组，即在括号内添加数组。

### 数组和字典的可选

#### 数组的类型推断

数组是可选的，数组内的值也是可选的，下面列举一些例子：

```swift
let arr1 = [1,2,3]				//数组不可选，值不可选
let arr2 = [1,2,nil]			//数组不可选，值可选
let arr3: [Int] = [1,2,3]		//数组不可选，值不可选
let arr4: [Int?] = [1,2,nil]	//数组不可选，值可选
let arr5: [Int]? = [1,2,3]		//数组可选，值不可选
let arr6: [Int?]? = [1,2,nil]	//数组可选，值可选
```

我们可以看到：

1. `arr1` 由于赋给的值都非空，所以数组值被类型推断为不可选。这和 `arr3` 同一个意思。
2. `arr2` 由于赋给的值有空，所以数组值被推断为可选。这和 `arr4` 同一个意思。
3. `arr5` 手动设置数组本生是可选的.
4. `arr6` 手动设置数组和数组值为可选的

上面对应的调用方式如下：

```swift
var sum1 = arr1[0] + 2
var sum2 = arr2[0]! + 2
var sum3 = arr3[0] + 2
var sum4 = arr4[0]! + 2
var sum5 = arr5![0] + 2
var sum6 = arr6![0]! + 2
```

1. `arr1` 中的值是非可选的，所以直接取出计算
2. `arr2` 中的值是可选的，所以取出后要强制解析然后才能计算
3. `arr3` 同 `arr1`,`arr4` 同 `arr2`
4. `arr5` 由于数组是可选的，所以要在 `arr` 后强制解析一下
5. `arr6` 由于数组和值都是可选的，所以 `arr` 以及 `[0]` 后都要强制解析



#### 字典的类型推断

字典也是可以进行类型推断的：

```swift
let dic1 = [1:1,2:2,3:3]
let dic2: [Int:Int] = [1:1,2:2,3:3]
let dic3: [Int:Int]? = [1:1,2:2,3:3]
```

可以看到字典中只有这三种写法。这是因为 dic 中如果值为 nil，那么就会将键删除，所以不存在字典值的可选的情况。虽然字典不存在值的可选，但是使用的时候可能获取不到对应的键值对，所以使用的时候需要使用强制解析：

```swift
var sum1 = dic1[1]! + 2
var sum3 = dic3![1]! + 2
```

上面的例子中，可能字典中不包含键为 `1` 的键值对，所以取出可能为空，这就需要强制解析了。这就涉及到了一个关于可选的原则：**如果要对一个可能为空的值操作，先将其强制解析**。

注意字典里获取不存在的键，是能返回 nil 的，所以返回值是个可选类型，要强制解析；而如果越界访问数组，不是返回 nil，直接抛出异常。


