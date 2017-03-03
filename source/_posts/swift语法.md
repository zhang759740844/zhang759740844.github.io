title: Swift 3.1 语法学习
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

可选可以用在一个值可能缺失的情况下。其实就是可能为基本类型，也可能为空。比如在强制类型转化中：

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

可选绑定可以用在 if 和 while 语句中来对可选的值进行判断，并把值赋给一个常量或者变量，来个例子：

```swift
if let actualNumber = Int(possibleNumber) {
    print("\"\(possibleNumber)\" has an integer value of \(actualNumber)")
} else {
    print("\"\(possibleNumber)\" could not be converted to an integer")
}
// Prints ""123" has an integer value of 123"
```

这个表示，如果 `Int(possibleNumber)` 转换后有值，那么赋值给 `actualNumber`，走成功的分支，否则走失败的分支。注意，这里的 `actualNumber` 就不需要加 `!` 了，因为由于可选绑定，它已经不是一个可选类型了。

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

Swift 中，如果创建了一个新的字符串，那么当其进行常量、变量赋值操作或在函数/方法中传递时，都会对已有字符串值创建新副本，并对该新副本进行传递或赋值。而在 oc 中，由于 `NSString` 是不可变的，所以所有操作都是对 `NSString` 实例的一个引用。

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

和 String 类似，如果你把一个集合赋给一个 `var` 类型的变量，那么你可以动态地对集合操作；如果你把一个集合赋给一个 `let` 的常量，那么这个集合就是不可变的（包括大小和内容）。

###  数组

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

上面的操作主要还是为了明确数组的类型。一个不明确的类型的数组声明，比如 `var someInts = []`是不合法的。但是如果数组本身的类型已经明确了，就可以直接用 `[]` 置空了：

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

通过 `insert(_:at:)` 在指定位置插入：

```swift
shoppingList.insert("Maple Syrup", at: 0)
// shoppingList now contains 7 items
// "Maple Syrup" is now the first item in the list
```

通过 `remove(_:at:)` 删除指定位置数组内容，并返回**删除的那个内容**（如果你用不到返回值，可以忽略）：

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

## 控制流

### For-In 循环

`for-In` 循环之前也接触过了。可以使用区间操作符控制循环次数：

```swift
for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}
// 1 times 5 is 5
// 2 times 5 is 10
// 3 times 5 is 15
// 4 times 5 is 20
// 5 times 5 is 25
```

上面的例子中， `index` 是一个自动被设置的值，不需要自己声明。

如果不需要循环的次数，可以用下划线 `_` 来替代 `index`。

```swift
let base = 3
let power = 10
var answer = 1
for _ in 1...power {
    answer *= base
}
print("\(base) to the power of \(power) is \(answer)")
// Prints "3 to the power of 10 is 59049"
```

### While 循环

#### while

while 的形式：

> while condition{
>
> ​	statements
>
> }

就是 condition 的时候没有括号，其他没有特别的地方。

#### Repeat-While

和其他语言的  do-while 类似，只不过换成了 repeat：

> repeat {
>
> ​	statements
>
> } while condition

### 条件语句

#### if

if 之前也用到过许多次了，和其他语言一样，注意判断条件不加括号。else 以及 else if 类似。

#### switch

switch 基本用法也差不多，直接看个例子：

```swift
let someCharacter: Character = "z"
switch someCharacter {
case "a":
    print("The first letter of the alphabet")
case "z":
    print("The last letter of the alphabet")
default:
    print("Some other character")
}
// Prints "The last letter of the alphabet"
```

#### 没有隐性掉入

相比而言，没有 `break`，匹配一个后，直接返回。也可以一条里面匹配多个，用逗号隔开：

```swift
let someCharacter: Character = "z"
switch someCharacter {
case "a","z":
    print("The first letter of the alphabet")
case "z":
    print("The last letter of the alphabet")
default:
    print("Some other character")
}
// Prints "The first letter of the alphabet"
```

这样就匹配到了第一个情况。

另外，每个 case 的主干都需要至少一句可执行语句，不能只有 case：

```swift
let anotherCharacter: Character = "a"
switch anotherCharacter {
case "a": // Invalid, the case has an empty body
case "A":
    print("The letter A")
default:
    print("Not the letter A")
}
// This will report a compile-time error.
```

#### 范围匹配

在 case 中可以写一个范围：

```swift
let approximateCount = 62
let countedThings = "moons orbiting Saturn"
var naturalCount: String
switch approximateCount {
case 0:
    naturalCount = "no"
case 1..<5:
    naturalCount = "a few"
case 5..<12:
    naturalCount = "several"
case 12..<100:
    naturalCount = "dozens of"
case 100..<1000:
    naturalCount = "hundreds of"
default:
    naturalCount = "many"
}
print("There are \(naturalCount) \(countedThings).")
// Prints "There are dozens of moons orbiting Saturn."
```

(`1..<5` 这种写法也是蛋疼 )

#### 元组

匹配的时候还可以匹配元组，用下划线 `_` 来匹配任意字符。当元组完全相等时，进入 case：

```swift
let somePoint = (1, 1)
switch somePoint {
case (0, 0):
    print("(0, 0) is at the origin")
case (_, 0):
    print("(\(somePoint.0), 0) is on the x-axis")
case (0, _):
    print("(0, \(somePoint.1)) is on the y-axis")
case (-2...2, -2...2):
    print("(\(somePoint.0), \(somePoint.1)) is inside the box")
default:
    print("(\(somePoint.0), \(somePoint.1)) is outside of the box")
}
// Prints "(1, 1) is inside the box"
```

#### 值绑定

在前面的基础上，前面用 `_` 代替任意值。如果在 case 中需要用到这个值怎么办呢？用 let 声明一个：

```swift
let anotherPoint = (2, 0)
switch anotherPoint {
case (let x, 0):
    print("on the x-axis with an x value of \(x)")
case (0, let y):
    print("on the y-axis with a y value of \(y)")
case let (x, y):
    print("somewhere else at (\(x), \(y))")
}
// Prints "on the x-axis with an x value of 2"
```

这里面 `let(x,y)` 和 `(let x,let y)` 是一样的。

#### where

switch 的 case 能使用 where 子句来进一步判断条件。 

```swift
let yetAnotherPoint = (1, -1)
switch yetAnotherPoint {
case let (x, y) where x == y:
    print("(\(x), \(y)) is on the line x == y")
case let (x, y) where x == -y:
    print("(\(x), \(y)) is on the line x == -y")
case let (x, y):
    print("(\(x), \(y)) is just some arbitrary point")
}
// Prints "(1, -1) is on the line x == -y"
```

三个 switch 的 case 声明了占位常量 x 和 y，临时占用 point 中元组值。这些常量作为 where 子句的一部分，用来创建动态的筛选。只有当 where 子句的条件结果为 true，Switch 的 case 则会匹配现有 point 的值。

（不觉得这样写 case 很方便。还不如 if-else 呢）

### 控制转移声明

控制转移声明包括：

- continue
- break
- fallthrough
- return
- throw

除了 `fallthrough` 其他都差不多。这里要注意一下在 switch 中使用的 `break`。由于 switch 中的 case 里的执行语句不能为空，所以如果匹配到一种情况不需要操作，可以直接用 `break`:

```swift
let numberSymbol: Character = "三"  // Chinese symbol for the number 3
var possibleIntegerValue: Int?
switch numberSymbol {
case "1", "١", "一", "๑":
    possibleIntegerValue = 1
case "2", "٢", "二", "๒":
    possibleIntegerValue = 2
case "3", "٣", "三", "๓":
    possibleIntegerValue = 3
case "4", "٤", "四", "๔":
    possibleIntegerValue = 4
default:
    break
}
```

#### Fallthrough

Swift 中在匹配成功后就不会掉入下一个 case 中。如果你就想要掉入的话，那么在执行语句后加上 `fallthrough` 即可：

```swift
let integerToDescribe = 5
var description = "The number \(integerToDescribe) is"
switch integerToDescribe {
case 2, 3, 5, 7, 11, 13, 17, 19:
    description += " a prime number, and also"
    fallthrough
default:
    description += " an integer."
}
print(description)
// Prints "The number 5 is a prime number, and also an integer."
```

特别注意：调用 `fallthrough` 后，**不检查 case 里的条件，会直接掉入下一个 case **。所以这里面直接执行了 default 的代码。

#### 标签声明

主要诱因是循环与 switch 的嵌套。有时候，你需要跳出外层循环或者 switch，但是由于嵌套，你不得不 `break` 或者 `continue` 好几次。那么，标签声明为循环和 switch 提供了一个标记，这样在循环或者 switch 内部，可以轻松的终止外部的循环或 switch。

```swift
gameLoop: while square != finalSquare {
    diceRoll += 1
    if diceRoll == 7 { diceRoll = 1 }
    switch square + diceRoll {
    case finalSquare:
        // diceRoll will move us to the final square, so the game is over
        break gameLoop
    case let newSquare where newSquare > finalSquare:
        // diceRoll will move us beyond the final square, so roll again
        continue gameLoop
    default:
        // this is a valid move, so find out its effect
        square += diceRoll
        square += board[square]
    }
}
print("Game over!")
```

这样，通过 `break gameLoop` 和 `continue gameLoop` 就可以方便的跳出整个循环。

### Guide 声明

这个是 Swift 3 的特性，和 if 类似。后面跟着一个 else，先看个例子：

```swift
func greet(person: [String: String]) {
    guard let name = person["name"] else {
        return
    }
    
    print("Hello \(name)!")
    
    guard let location = person["location"] else {
        print("I hope the weather is nice near you.")
        return
    }
    
    print("I hope the weather is nice in \(location).")
}
 
greet(person: ["name": "John"])
// Prints "Hello John!"
// Prints "I hope the weather is nice near you."
greet(person: ["name": "Jane", "location": "Cupertino"])
// Prints "Hello Jane!"
// Prints "I hope the weather is nice in Cupertino."
```

当 `guard` 满足的时候代码直接往后走，如果条件不满足，那么执行 else 内的代码。其实逻辑和 if 是一样的。但是为什么要弄出这么个东西呢？为了让代码可读性更高。

### 检查 API 是否可用

Swift 3 中提供了检查 API 是否可用的方法：

```swift
if #available(iOS 10, macOS 10.12, *) {
    // Use iOS 10 APIs on iOS, and use macOS 10.12 APIs on macOS
} else {
    // Fall back to earlier iOS and macOS APIs
}
```

表示在 iOS 10，macOS 10.12，以及任意其他平台可用。`*` 表示任意其他平台。

## 函数

### 定义一个函数

直接上例子：

```swift
func greet(person: String) -> String {
    let greeting = "Hello, " + person + "!"
    return greeting
}
```

这个函数名为 ` greet(person:)`。这个函数输入一个 `String` 类型叫做 `person` 的值，输出一段字符串。函数以 `func` 为前缀，`->` 指定的返回类型。

可以调用 `print(_:separator:terminator:) ` 方法去打印上面函数的返回值：

```swift
print(greet(person: "Anna"))
// Prints "Hello, Anna!"
print(greet(person: "Brian"))
// Prints "Hello, Brian!"
```

这里的 `print(_:separator:terminator:)` 方法第一个入参没有名字，后面两个入参由于有默认值，因此可选。具体将在下面说到。

### 函数的参数和返回值

#### 没有参数

示例：

```swift
func sayHelloWorld() -> String {
    return "hello, world"
}
print(sayHelloWorld())
// Prints "hello, world"
```

#### 多个参数

函数可以有多个输入参数，把他们写到函数的括号内，并用逗号加以分隔：

```swift
func greet(person: String, alreadyGreeted: Bool) -> String {
    if alreadyGreeted {
        return greetAgain(person: person)
    } else {
        return greet(person: person)
    }
}
print(greet(person: "Tim", alreadyGreeted: true))
// Prints "Hello again, Tim!"
```

#### 无返回值

没有 `return` 没有返回类型 `->`：

```swift
func greet(person: String) {
    print("Hello, \(person)!")
}
greet(person: "Dave")
// Prints "Hello, Dave!"
```

严格来说，其实无返回类型的函数还是返回了一个值，即使没有返回值定义。函数没有定义返回类型但返 回了一个 `void` 返回类型的特殊值。它是一个空的元组，可以写为`()`

#### 多个返回值

可以将返回类型设置为元组，来返回多个值：

```swift
func minMax(array: [Int]) -> (min: Int, max: Int) {
    var currentMin = array[0]
    var currentMax = array[0]
    for value in array[1..<array.count] {
        if value < currentMin {
            currentMin = value
        } else if value > currentMax {
            currentMax = value
        }
    }
    return (currentMin, currentMax)
}
```

注意 `->` 后面的返回类型书写方式：`(min: Int, max: Int) ` 。

由于在定义返回类型的时候，元组中的元素都已经被命名好了。所以我们通过 `.` 操作符就能拿到对应的元素：

```swift
let bounds = minMax(array: [8, -6, 2, 109, 3, 71])
print("min is \(bounds.min) and max is \(bounds.max)")
// Prints "min is -6 and max is 109"
```

#### 可选的元组返回类型

如果返回的元组可能没有值，那么可以使用可选的元组作为返回类型，来表示整个元组可以为 `nil`。在元组之后加上 `?` 来表示可选类型，例如 `(Int,Int)?` （注意不要写成 `(Int?,Int?)` 这个表示元组中的元素是可选的）。

如果是可选元组，那么一定要先做非空判断。否则当你想要取出元组中元素后，就会触发运行时错误。你可以通过可选绑定来判断是否为空：

```swift
if let bounds = minMax(array: [8, -6, 2, 109, 3, 71]) {
    print("min is \(bounds.min) and max is \(bounds.max)")
}
```

### 函数的 argument label 和 parameter name

这两个词我不知道怎么翻译。每个函数都有 argument label 和 parameter name。argument label 用在调用函数的时候；parameter name 用在方法体中的参数使用。默认情况下两者是相等的。在一个方法中，所有的 parameter name 都应该是独一无二的，因为在方法体中会用到；argument label 则不必都不同。 （如果看不懂啥意思，可以看下面的几个例子就明白了）。

```swift
func someFunction(firstParameterName: Int, secondParameterName: Int) {
    // In the function body, firstParameterName and secondParameterName
    // refer to the argument values for the first and second parameters.
}
someFunction(firstParameterName: 1, secondParameterName: 2)
```

上面是一个 argument laebl 和 parameter name 默认相等的例子。

#### 明确的 argument labels

现在考虑 argument labels 和 parameter name 不同的情况。可以将 argumentLabel 写在 parameterName 前面：

```swift
func someFunction(argumentLabel parameterName: Int) {
    // In the function body, parameterName refers to the argument value
    // for that parameter.
}
```

来看一个具体实例：

```swift
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
// Prints "Hello Bill!  Glad you could visit from Cupertino."
```

其中 `from` 是 argument labels，`hometown` 是 parameter name。注意在方法里用 `hometown`，在调用的时候用 `from`。

#### 省略的 argument labels

如果不想要 argument labels，可以用 `_` 代替，调用的时候就什么都不用写了：

```swift
func someFunction(_ firstParameterName: Int, secondParameterName: Int) {
    // In the function body, firstParameterName and secondParameterName
    // refer to the argument values for the first and second parameters.
}
someFunction(1, secondParameterName: 2)
```

注意，参数顺序还是不能错的。

#### 默认的参数值

在定义函数的时候可以在参数类型后加上默认值：

```swift
func someFunction(parameterWithoutDefault: Int, parameterWithDefault: Int = 12) {
    // If you omit the second argument when calling this function, then
    // the value of parameterWithDefault is 12 inside the function body.
}
someFunction(parameterWithoutDefault: 3, parameterWithDefault: 6) // parameterWithDefault is 6
someFunction(parameterWithoutDefault: 4) // parameterWithDefault is 12
```

如果有默认值了，那么在调用的时候这个参数就可以省略了。

推荐将有默认值的参数放在入参的后面。因为没有默认值的参数一般更重要。

#### 可变数量的入参

一个可变数量的入参可以传入零个或者更多的参数，在参数类型后面加上 `…` 就行了。这些入参将会以一个数组的形式在方法体中使用：

```swift
func arithmeticMean(_ numbers: Double...) -> Double {
    var total: Double = 0
    for number in numbers {
        total += number
    }
    return total / Double(numbers.count)
}
arithmeticMean(1, 2, 3, 4, 5)
// returns 3.0, which is the arithmetic mean of these five numbers
arithmeticMean(3, 8.25, 18.75)
// returns 10.0, which is the arithmetic mean of these three numbers
```

这里入参的数字将会以名为  `numbers` 的 `[Double]` 数组的形式传入。

这里有几个注意点：

1. 为什么不直接传一个数组呢？因为那样不直观
2. 上面的是省略了 argument labels 的情况。如果不省略，比如将 `_` 替换成 `to`，那么调用的时候改为 `arithmeticMean(to:1, 2, 3, 4, 5)` 即可。
3. 一个函数里最多只能有一个可变数量的入参。
4. 如果参数是可变的那么就没法设置默认值了。
5. 还有一种情形：`func arithmeticMean(_ numbers: Double...,_ anotherNumber: Double)`，这种情况下由于两个入参都是缺省的，所有传入的参数都被 `numbers` 接收，第二个参数无法接收参数。

#### 输入输出参数

一般函数内参数值的变化不会影响外部传入的变量的值。现在可以通过 `inout` 标记，使参数值的变化同步影响到外部值的变化。

```swift
func swapTwoInts(_ a: inout Int, _ b: inout Int) {
    let temporaryA = a
    a = b
    b = temporaryA
}
```

这个示例中，该函数交换输入的两个值。在返回类型前加上 `inout` 标记。调用方法的时候需要在输入参数前加上 `&`(其实就是 c 里面的引用嘛，只是多加个 inout 标记用来提示调用者)。使用示例：

```swift
var someInt = 3
var anotherInt = 107
swapTwoInts(&someInt, &anotherInt)
print("someInt is now \(someInt), and anotherInt is now \(anotherInt)")
// Prints "someInt is now 107, and anotherInt is now 3"
```

输入输出参数需要使用变量而不是常量，因为常量不能改变。

### 函数类型

上面有提到过，每个函数都有其函数类型，由输入参数类型和返回类型组成。比如：

```swift
func addTwoInts(_ a: Int, _ b: Int) -> Int {
    return a + b
}
func multiplyTwoInts(_ a: Int, _ b: Int) -> Int {
    return a * b
}
```

上面的方法的函数类型为 `(Int, Int) -> Int`。再比如一个没有入参和返回值的函数：

```swift
func printHelloWorld() {
    print("hello, world")
}
```

这个函数的类型是 `()->Void`

#### 使用函数类型

你可以定义一个常量或变量为一个函数类型，并指定适当的函数给该变量：

```swift
var mathFunction: (Int, Int) -> Int = addTwoInts
```

现在就可以像使用 `addTwoInts` 方法一样使用 `mathFunction` 了：

```swift
print("Result: \(mathFunction(2, 3))")
// Prints "Result: 5"
```

注意，只有函数类型匹配才能够将 `addTwoInts` 赋给 `mathFunction`。由于 `mathFunction` 和 `multiplyTwoInts` 类型也相同，所以可以继续对 `mathFunction` 赋值：

```swift
mathFunction = multiplyTwoInts
print("Result: \(mathFunction(2, 3))")
// Prints "Result: 6"
```

由于类型推断的作用，定义函数变量或者常量的时候可以不用写出函数类型：

```swift
let anotherMathFunction = addTwoInts
// anotherMathFunction is inferred to be of type (Int, Int) -> Int
```

#### 函数类型作为参数类型

可以将一个函数作为参数传入另一个函数。这使你预留了一个函数的某些方面的函数实现，让调用者提供的函数时被调用：

```swift
func printMathResult(_ mathFunction: (Int, Int) -> Int, _ a: Int, _ b: Int) {
    print("Result: \(mathFunction(a, b))")
}
printMathResult(addTwoInts, 3, 5)
// Prints "Result: 8"
```

这里 `mathFunction` 就是一个函数类型 `(Int,Int)->Int`。

#### 函数类型作为返回类型

函数的返回类型可以是一个函数类型，即返回一个函数。

例如有两个函数，函数类型都为 `(Int)->Int`:

```swift
func stepForward(_ input: Int) -> Int {
    return input + 1
}
func stepBackward(_ input: Int) -> Int {
    return input - 1
}
```

现在定义一个返回 `(Int)->Int` 类型的函数 `chooseStepFunction(backward:)`，注意返回类型的写法:

```swift
func chooseStepFunction(backward: Bool) -> (Int) -> Int {
    return backward ? stepBackward : stepForward
}
```

下面看看如何使用的：

```swift
var currentValue = 3
let moveNearerToZero = chooseStepFunction(backward: currentValue > 0)
// moveNearerToZero now refers to the stepBackward() function

print("Counting to zero:")
// Counting to zero:
while currentValue != 0 {
    print("\(currentValue)... ")
    currentValue = moveNearerToZero(currentValue)
}
print("zero!")
// 3...
// 2...
// 1...
// zero!

```

### 嵌套函数

迄今为止碰到的所有函数都是在全局范围里的函数，我们也可以将函数定义在函数内，作为嵌套函数。嵌套函数对外是隐藏的，但仍然可以调用和使用其内部的函数。

```swift
func chooseStepFunction(backward: Bool) -> (Int) -> Int {
    func stepForward(input: Int) -> Int { return input + 1 }
    func stepBackward(input: Int) -> Int { return input - 1 }
    return backward ? stepBackward : stepForward
}
var currentValue = -4
let moveNearerToZero = chooseStepFunction(backward: currentValue > 0)
// moveNearerToZero now refers to the nested stepForward() function
while currentValue != 0 {
    print("\(currentValue)... ")
    currentValue = moveNearerToZero(currentValue)
}
print("zero!")
// -4...
// -3...
// -2...
// -1...
// zero!
```

## 闭包

### 闭包表达式

#### sort 函数

Swift 提供了 `sort(by:)` 函数，会根据提供的闭包，将已知类型数组中的值进行排序。排序完成，函数会返回一个与原数组大小相同的新数组，该数组中包含已经正确排序的同类型元素。

`sort(by:)` 函数输入一个比较函数大小的方法：

```swift
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]

func backward(_ s1: String, _ s2: String) -> Bool {
    return s1 > s2
}
var reversedNames = names.sorted(by: backward)
// reversedNames is equal to ["Ewa", "Daniella", "Chris", "Barry", "Alex"]
```

#### 闭包表达式语法

上面是输入一个函数的方式，还可以创建一个闭包表达式。闭包表达式的语法如下：

> {(parameters) -> returntype in
>
> ​	statements
>
> }

所以可以将上面的排序方法改为如下：

```swift
reversedNames = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})
```

#### 根据上下文推断类型

因为闭包是**作为函数的参数传入**的，Swift 能推断出它的参数和返回值的类型(**注意，是只有作为参数传入的闭包，才能依据函数的参数类型推断，省略闭包的参数类型的定义。如果是自己单独定义的一个闭包，不能进行上下文推断类型。**)，所以可以省略类型：

```swift
reversedNames = names.sorted(by: { s1, s2 in return s1 > s2 } )
```

实际上任何情况下，通过内联闭包表达式构造的闭包作为参数传递给函数时，都可以推断出闭包的参数和返回值类型，这意味着您几乎不需要利用完整格式构造任何内联闭包。然而，你也可以使用明确的类型，如果你想它避免读者阅读可能存在的歧义，这样还是值得鼓励的。

#### 单行表达式省略

单行表达式闭包可以通过隐藏 `return` 关键字来隐式返回单行表达式的结果，如上版本的例子可以改写为：

```swift 
reversedNames = names.sorted(by: { s1, s2 in s1 > s2 } )
```

#### 参数名简写

Swift 为内联函数提供了参数名称简写功能。可以直接用 $0,$1,$2 等名字来引用的闭包的参数的值。此时，`in` 关键字可以被省略。在这个例子中，$0 和 $1 表示闭包中第一个和第二个 String 类型的参数。

```swift
reversedNames = names.sorted(by: { $0 > $1 } )
```

### 尾部闭包

如果定义的函数的最后一个参数是一个闭包，调用的时候可以使用尾部闭包(Trailing Closures 不知道怎么翻译好就解释为尾部闭包了)。注意书写语法：

```swift
func someFunctionThatTakesAClosure(closure: () -> Void) {
    // function body goes here
}
 
// 没有使用尾部闭包的情况
someFunctionThatTakesAClosure(closure: {
    // closure's body goes here
})
 
// 使用尾部闭包的情况 
someFunctionThatTakesAClosure() {
    // trailing closure's body goes here
}
```

可以看到很明显的区别。没有使用尾部闭包的时候是非常正规的函数调用。如果是尾部闭包的情况，那么就单独拿出来放到外面。

比如上面的 `sorted(by:)` 方法可以改成尾部闭包的形式：

```swift
reversedNames = names.sorted() { $0 > $1 }
```

如果整个函数的参数只有这一个入参，那么还可以直接省略这个括号(虽然没有歧义，但是总感觉这种语法糖不太好)：

```swift
reversedNames = names.sorted { $0 > $1 }
```

再举一个典型的例子：在 Swift 中的数组类型中，有一个 `map(_:)` 方法，这个方法输入一个处理函数，对数组中的每个元素进行处理，返回一个处理过的新数组：

```swift
let digitNames = [
    0: "Zero", 1: "One", 2: "Two",   3: "Three", 4: "Four",
    5: "Five", 6: "Six", 7: "Seven", 8: "Eight", 9: "Nine"
]
let numbers = [16, 58, 510]

let strings = numbers.map { (number) -> String in
    var number = number
    var output = ""
    repeat {
        output = digitNames[number % 10]! + output
        number /= 10
    } while number > 0
    return output
}
// strings is inferred to be of type [String]
// its value is ["OneSix", "FiveEight", "FiveOneZero"]
```

这里面 `digitNames[number % 10]!` 这个 `!` 可以复习一下。上面讲到过，用来强制解析。通过数组下标返回的值都是可选类型的。所以要强制解析成字符串。

### 捕获值

闭包可以在捕获其所定义的上下文中的变量和常量。即使定义这些常量和变量的原作用域已经不存在，闭包仍然可以在闭包函数体内引用和修改这些值。

下面举一个关于嵌套函数的例子。嵌套函数可以捕获其外部函数所有的参数以及定义的常量和变量：

```swift
func makeIncrementer(forIncrement amount: Int) -> () -> Int {
    var runningTotal = 0
    func incrementer() -> Int {
        runningTotal += amount
        return runningTotal
    }
    return incrementer
}

// 使用:
let incrementByTen = makeIncrementer(forIncrement: 10)
incrementByTen()
// returns a value of 10
incrementByTen()
// returns a value of 20
incrementByTen()
// returns a value of 30
```

`incrementer` 函数中并没有保存 `amount` 和 `runningTotal`。`incrementor` 实际上捕获并存储了该变量的一个副本，而该副本随着 `incrementor` 一同被存储。

如果新建了一个新的 `incrementer` ，其会有一个属于自己的独立的 `runningTotal` 变量的引用：

```swift
let incrementBySeven = makeIncrementer(forIncrement: 7)
incrementBySeven()
// returns a value of 7
incrementByTen()
// returns a value of 40
```



### 闭包是引用类型

上面的例子中，`incrementBySeven` 和 `incrementByTen` 是常量，但是这些常量指向的闭包仍然可以增加其捕获的变量值。 这是因为函数和闭包都是引用类型。

无论您将函数/闭包赋值给一个常量还是变量，您实际上都是将常量/变量的值设置为对应函数/闭包的引用。 上面的例子中，`incrementByTen` 指向闭包的引用是一个常量，而并非闭包内容本身。

这也意味着如果您将闭包赋值给了两个不同的常量/变量，两个值都会指向同一个闭包：

```swift
let alsoIncrementByTen = incrementByTen
alsoIncrementByTen()
// returns a value of 50
```

### 逃逸闭包

当一个闭包**作为参数**传到另一个函数中，并且这个闭包在函数返回后才调用（比如在函数中执行一个异步回调，回调时才执行这个闭包），我们称该闭包从函数中逃逸。需要在函数名前标注 `@escaping`，用来表示这个闭包是允许逃逸出函数的。

下面模拟一下情景：将 `completionHandler` 参数保存在外部的一个数组中，此时必须要标记 `@escaping` 否则产生编译错误：

```swift
var completionHandlers: [() -> Void] = []
func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
    completionHandlers.append(completionHandler)
}
```

将闭包标记为逃逸闭包，就必须在闭包中显式地引用 `self`。看下面的示例：

```swift
func someFunctionWithNonescapingClosure(closure: () -> Void) {
    closure()
}

class SomeClass {
    var x = 10
    func doSomething() {
        someFunctionWithEscapingClosure { self.x = 100 }
        someFunctionWithNonescapingClosure { x = 200 }
    }
}

let instance = SomeClass()
instance.doSomething()
print(instance.x)
// 打印出 "200"

completionHandlers.first?()
print(instance.x)
// 打印出 "100"
```

其中，逃逸的闭包没有立刻执行，所以 `instance.x` 先被设置成了 100，然后闭包执行了后，才被设置成 200。

### 自动闭包

自动闭包是一种闭包的简写方式。不接受任何参数，当他被调用时，会返回被包装在其中的表达式的值。这种便利语法让你能够省略闭包的花括号，用一个普通的表达式来代替显式的闭包。

来看个例子就明白了，先看一下不用自动闭包的情况：

```swift
// customersInLine is ["Alex", "Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: { customersInLine.remove(at: 0) } )
// 打印出 "Now serving Alex!"
```

上面这种情况的函数调用是正常的打开方式。下面来看下自动闭包的使用：

```swift
func serve(customer customerProvider: @autoclosure () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: customersInLine.remove(at: 0))
// 打印出 "Now serving Ewa!"
```

上面用了 `autoclosure` 标记后，下面的闭包可以不用括号。(**这玩意会有人用？毫无意义。就当记录一下吧**)

## 枚举

### 枚举语法

使用`enum`关键词来创建枚举并且把它们的整个定义放在一对大括号内：

```swift
enum SomeEnumeration {
    // 枚举定义放在这里
}
```

例子:

```swift
enum CompassPoint {
    case North
    case South
    case East
    case West
}
```

注意每一个 `case` 来定义一个新的枚举成员。如果出现在同一行上，需要用逗号隔开：

```swift
enum Planet {
    case Mercury, Venus, Earth, Mars, Jupiter, Saturn, Uranus, Neptune
}
```

和 oc 不同，这里的枚举值不会被隐式的赋值为 0，1，2，3。这些枚举成员本身就是完备的值，比如最上面的这些值的类型是已经明确定义好的 `CompassPoint` 类型。

使用方式：

```swift
var directionToHead = CompassPoint.West
```

`directionToHead`的类型可以在它被`CompassPoint`的某个值初始化时推断出来。一旦`directionToHead`被声明为`CompassPoint`类型，你可以使用更简短的点语法将其设置为另一个`CompassPoint`的值：

```swift
directionToHead = .East
```

### 使用 switch 枚举

可以使用 `switch` 匹配枚举值：

```swift
directionToHead = .South
switch directionToHead {
    case .North:
        print("Lots of planets have a north")
    case .South:
        print("Watch out for penguins")
    case .East:
        print("Where the sun rises")
    case .West:
        print("Where the skies are blue")
    default:
    	print("Not a safe place for humans")
}
// 输出 "Watch out for penguins”
```

### 关联值

有些时候枚举值会需要存储一些关联值以方便使用，比如：

```swift
enum Barcode {
    case UPCA(Int, Int, Int, Int)
    case QRCode(String)
}
```

表示 `UPCA` 具有 `(Int，Int，Int，Int)` 的关联值，`QRCode` 具有 `String` 的关联值。使用：

```swift
var productBarcode = Barcode.UPCA(8, 85909, 51226, 3)
productBarcode = .QRCode("ABCDEFGHIJKLMNOP")
```

类型推断为 `Barcode`，关联值只是附加信息，便于存储一些必要信息。比如在使用 switch 语句时，可以将关联值提取出来，就可以在执行语句中使用。可以在`switch`的 case 分支代码中提取每个关联值作为一个常量（用`let`前缀）或者作为一个变量（用`var`前缀）来使用：

```swift
switch productBarcode {
case .UPCA(let numberSystem, let manufacturer, let product, let check):
    print("UPC-A: \(numberSystem), \(manufacturer), \(product), \(check).")
case .QRCode(let productCode):
    print("QR code: \(productCode).")
}
// 输出 "QR code: ABCDEFGHIJKLMNOP."
```

为了简洁，可以将`let`或者`var` 提取出来：

```swift
switch productBarcode {
case let .UPCA(numberSystem, manufacturer, product, check):
    print("UPC-A: \(numberSystem), \(manufacturer), \(product), \(check).")
case let .QRCode(productCode):
    print("QR code: \(productCode).")
}
// 输出 "QR code: ABCDEFGHIJKLMNOP."
```

### 原始值

原始值的定义和 oc 中枚举的效果很像。可以为每个枚举成员定义一个默认值。这些默认值的类型必须相同：

```swift
enum ASCIIControlCharacter: Character {
    case Tab = "\t"
    case LineFeed = "\n"
    case CarriageReturn = "\r"
}
```

其中，将枚举类型定义为字符串类型。原始值还可以是字符，任意整形或浮点型值。

注意，原始值和关联值是不同的。原始值是定义枚举时被预先填充的值。对于一个特定的枚举成员，原始值始终不变。关联值是创建一个基于枚举成员的常量或变量时才设置的值，枚举成员的关联值可以变化。

#### 原始值的隐式赋值

使用证书或者字符串作为原始值枚举时，不需要显式赋值，Swift 会自动赋值：

```swift
enum Planet: Int {
    case Mercury = 1, Venus, Earth, Mars, Jupiter, Saturn, Uranus, Neptune
}
```

在上面的例子中，`Plant.Mercury`的显式原始值为`1`，`Planet.Venus`的隐式原始值为`2`。

当使用字符串作为枚举类型的原始值时，每个枚举成员的隐式原始值为该枚举成员的名称：

```swift
enum CompassPoint: String {
    case North, South, East, West
}
```

上面例子中，`CompassPoint.South`拥有隐式原始值`South`，即就是其本生。可以使用枚举成员的`rawValue`属性可以访问该枚举成员的原始值：

```swift
let earthsOrder = Planet.Earth.rawValue
// earthsOrder 值为 3

let sunsetDirection = CompassPoint.West.rawValue
// sunsetDirection 值为 "West"
```

**这里拿出了 `rawValue`，那么 `earthsOrder` 和`sunsetDirection` 就是明确的值了，而不是枚举类型。**

#### 使用原始值初始化枚举实例

如果在定义枚举类型的时候使用了原始值，那么将会自动获得一个初始化方法，这个方法接收一个叫做`rawValue`的参数，参数类型即为原始值类型，返回值则是枚举成员或`nil`。你可以使用这个初始化方法来创建一个新的枚举实例。

比如利用原始值`7`创建了枚举成员`Uranus`：

```swift
let possiblePlanet = Planet(rawValue: 7)
// possiblePlanet 类型为 Planet? 值为 Planet.Uranus
```

原始值构造器总是返回一个*可选*的枚举成员，因为可能没有对应的枚举类型。在上面的例子中，`possiblePlanet`是`Planet?`类型。

### 递归枚举

没啥用，暂时不看了。说不定以后就删了



## 类和结构体

### 定义语法

使用 `class` 和 `struct` 分别表示类和结构体。示例如下：

```swift
struct Resolution {
    var width = 0
    var height = 0
}
class VideoMode {
    var resolution = Resolution()
    var interlaced = false
    var frameRate = 0.0
    var name: String?
}
```

这里面可以注意一点。就是对于变量进行了初始化。这样有什么好处呢？可以使 swift 进行类型推断出当前变量的类型。在 **swift 中，每个变量的类型都必须是确定的。**

### 类和结构体实例

创建类和结构体实例的语法相似：

```swift
let someResolution = Resolution()
let someVideoMode = VideoMode()
```

通过这种方式所创建的类或者结构体实例，其属性均会被初始化为默认值。更多构造过程在后面将会更详细的讨论。

### 属性访问

通过 `.` 语法，可以访问实例的属性。与 oc 不同的是，Swift 允许直接设置结构体属性的子属性。反正通过 `.` 什么都能拿到就是了。

### 结构体类型的成员逐一构造器

所有**结构体（特指结构体）**都有一个自动生成的*成员逐一构造器*，用于初始化新结构体实例中成员的属性。新实例中各个属性的初始值可以通过属性的名称传递到成员逐一构造器之中：

```swift
let vga = Resolution(width:640, height: 480)
```

与结构体不同，类实例没有默认的成员逐一构造器。详细见后面。

### 结构体和枚举是值类型

什么是值类型？就是我们所说的值传递和引用传递中的值传递。指在被赋给一个变量常量或者传递给一个函数的时候，值会被拷贝。

所有的基本类型都是值拷贝这和以前毫无异议。但是在 Swift 中，**字符串、数组和字典也是值拷贝**，这和 oc 中很不相同。这意味着数组和字典中的所有元素都会被拷贝一份。其实是因为**它们底层都是以结构体的形式实现的**。

```swift
let hd = Resolution(width: 1920, height: 1080)
var cinema = hd
```

例如上面的例子，`cinema` 和 `hd` 相同，但其实在内存中是两个不同的对象。修改其中一个的值不会改变另一个实例相应属性的值。**枚举同样**。

### 类是引用类型

这点没啥不同的。就是不同常量或者变量指向内存上的相同地址。



### 恒等运算符

Swift 中内建了两个恒等运算符：

- 等价于（`===`）
- 不等价于 （`!==`）

注意这和“等于” `==` 有什么区别呢？

- "等价于"表示两个**类类型**(注意只能用在类类型中)的常量或者变量引用同一个类实例。
- “等于”表示两个实例的值“相等”或“相同”。

也就是说 `===` 为 `true` ，那么两个类实例必然指向同一块内存地址。`==` 为 `true` 则只要类内属性相同即可。（oc 中的 `==` 就是这里的 `===`，比较的是指针地址。）



### 指针

一个引用某个引用类型实例的 Swift 常量或者变量，与 C 语言中的指针类似，但是并不直接指向某个内存地址，也不要求你使用星号（`*`）来表明你在创建一个引用。Swift 中的这些引用与其它的常量或变量的定义方式相同。(反正就是简写了)



### 类和结构体的选择

结构体实例总是通过值传递，类实例总是通过引用传递。这意味两者适用不同的任务。

其实大部分数据构造都是用类，而非结构体。只有在数据结构非常简单的时候用结构体。



### 字符串、数组、字典的赋值和复制行为

上面也说过了 Swift 中的 `String`，`Array`和`Dictionary`类型均以结构体的形式实现。这意味着被赋值给新的常量或变量，或者被传入函数或方法中时，它们的值会被拷贝。

Objective-C 中`NSString`，`NSArray`和`NSDictionary`类型均以类的形式实现，而并非结构体。它们在被赋值或者被传入函数或方法时，不会发生值拷贝，而是传递现有实例的引用。

不过不用担心值拷贝会影响新能，Swift 中有优化。

## 属性

### 存储属性

类和结构体中用 `var` 或者 `let` 修饰的就是存储属性：

```swift
struct FixedLengthRange {
    var firstValue: Int
    let length: Int
}
var rangeOfThreeItems = FixedLengthRange(firstValue: 0, length: 3)
// 该区间表示整数0，1，2
rangeOfThreeItems.firstValue = 6
// 该区间现在表示整数6，7，8
```

#### 常量结构体的存储属性

**如果创建了一个结构体的实例并将其赋值给一个常量，则无法修改该实例的任何属性，即使有属性被声明为变量也不行**：

```swift
let rangeOfFourItems = FixedLengthRange(firstValue: 0, length: 4)
// 该区间表示整数0，1，2，3
rangeOfFourItems.firstValue = 6
// 尽管 firstValue 是个变量属性，这里还是会报错
```

因为 `rangeOfFourItems` 被声明成了常量（用 `let` 关键字），即使 `firstValue` 是一个变量属性，也无法再修改它了。

**这种行为是由于结构体（struct）属于*值类型*。当值类型的实例被声明为常量的时候，它的所有属性也就成了常量。**属于*引用类型*的类（class）则不一样。把一个引用类型的实例赋给一个常量后，仍然可以修改该实例的变量属性。

**因此注意，数组字典等如果设置为 `let` ，那么里面的内容也是不能改的。**

#### 延迟存储属性

指当第一次被调用的时候才会计算其初始值的属性。在属性声明前使用 `lazy` 来标识。注意，延迟存储属性必须被声明为 `var`。

```swift
class DataImporter {
    /*
    DataImporter 是一个负责将外部文件中的数据导入的类。
    这个类的初始化会消耗不少时间。
    */
    var fileName = "data.txt"
    // 这里会提供数据导入功能
}

class DataManager {
    lazy var importer = DataImporter()
    var data = [String]()
    // 这里会提供数据管理功能
}

let manager = DataManager()
manager.data.append("Some data")
manager.data.append("Some more data")
// DataImporter 实例的 importer 属性还没有被创建
```

上面这个类中，`DataImporter` 是一个很费时的操作。所以设置为 `lazy`，只有在第一次访问到的时候才会被创建。



### 计算属性

计算属性不直接存储值，而是提供一个 getter 和一个可选的 setter，来间接获取和设置其他属性或变量的值。**其实就是每次点到这个属性的时候都会再计算一遍，适用于会根据其他值变化的属性，这样就不用每次用到的时候专门调用一个处理方法了。而是由系统直接调用了 getter 方法**

```swift
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
var square = Rect(origin: Point(x: 0.0, y: 0.0),
    size: Size(width: 10.0, height: 10.0))
let initialSquareCenter = square.center
square.center = Point(x: 15.0, y: 15.0)
print("square.origin is now at (\(square.origin.x), \(square.origin.y))")
// 输出 "square.origin is now at (10.0, 10.0)”
```

上面这个矩形的类，通过 `origin` 和 `size` 来计算出 `center`。其中 setter 方法的 `newCenter` 由于类型推断，默认为 `Point` 类型，就不用再写明类型了。

#### 便捷 setter 声明

由于 `setter` 函数必然要传入一个新值，所以 Swift 定义了一个默认名称 `newValue`。所以可以采取简略的形式：

```swift
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set {
            origin.x = newValue.x - (size.width / 2)
            origin.y = newValue.y - (size.height / 2)
        }
    }
}
```

#### 只读计算属性

只有 getter 没有 setter 的计算属性就是*只读计算属性*。只读计算属性总是返回一个值，可以通过点运算符访问，但不能设置新的值。

如果是只读计算属性，那么连 `get` 关键字都可以扔掉了：

```swift
struct Cuboid {
    var width = 0.0, height = 0.0, depth = 0.0
    var volume: Double {
        return width * height * depth
    }
}
let fourByFiveByTwo = Cuboid(width: 4.0, height: 5.0, depth: 2.0)
print("the volume of fourByFiveByTwo is \(fourByFiveByTwo.volume)")
// 输出 "the volume of fourByFiveByTwo is 40.0"
```

### 属性观察器

可以监听属性值的变化（可以为除了延迟存储属性之外的其他存储属性添加属性观察器，因为可以通过 setter 方法直接监控）。

提供了两个属性观察器：

- `willSet` 新值被设置前调用
- `didSet` 新值被设置后调用

`willSet` 接受新的属性值作为常量传入，可以自己指定这个参数的名称。如果不指定，默认名称为 `newValue`。

`didSet` 将旧的属性值传入，不接受自定义参数名，默认参数名为 `oldValue`。如果在 `didSet` 方法中再次对该属性赋值，那么新值会覆盖旧的值。

```swift
class StepCounter {
    var totalSteps: Int = 0 {
        willSet(newTotalSteps) {
            print("About to set totalSteps to \(newTotalSteps)")
        }
        didSet {
            if totalSteps > oldValue  {
                print("Added \(totalSteps - oldValue) steps")
            }
        }
    }
}
let stepCounter = StepCounter()
stepCounter.totalSteps = 200
// About to set totalSteps to 200
// Added 200 steps
stepCounter.totalSteps = 360
// About to set totalSteps to 360
// Added 160 steps
stepCounter.totalSteps = 896
// About to set totalSteps to 896
// Added 536 steps
```

如果将属性通过 in-out 方式传入函数，`willSet` 和 `didSet` 也会调用。这是因为 in-out 参数采用了拷入拷出模式：即在函数内部使用的是参数的 copy，函数结束后，又对参数重新赋值。

### 全局变量和局部变量

全局变量是在函数、方法、闭包或任何类型之外定义的变量。局部变量是在函数、方法或闭包内部定义的变量。

全局的常量或变量都是延迟计算的，跟延迟存储属性相似，不同的地方在于，全局的常量或变量不需要标记`lazy`修饰符。
局部范围的常量或变量从不延迟计算。

### 类型属性

实例属性属于一个特定类型的实例，每创建一个实例，实例都拥有属于自己的一套属性值，实例之间的属性相互独立。类似于静态变量或常量。

跟实例的存储型属性不同，必须给存储型类型属性指定默认值，因为类型本身没有构造器，也就无法在初始化过程中使用构造器给类型属性赋值。
存储型类型属性是延迟初始化的，它们只有在第一次被访问的时候才会被初始化。即使它们被多个线程同时访问，系统也保证只会对其进行一次初始化，并且不需要对其使用 `lazy` 修饰符。

####  类型属性语法

在 C 或 Objective-C 中，与某个类型关联的静态常量和静态变量，是作为全局（*global*）静态变量定义的。但是在 Swift 中，类型属性是作为类型定义的一部分写在类型最外层的花括号内，因此它的作用范围也就在类型支持的范围内。

使用关键字 `static` 来定义类型属性。在为类定义计算型类型属性时，可以改用关键字 `class` 来支持子类对父类的实现进行重写。下面的例子演示了存储型和计算型类型属性的语法：

```swift
struct SomeStructure {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 1
    }
}
enum SomeEnumeration {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 6
    }
}
class SomeClass {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 27
    }
    class var overrideableComputedTypeProperty: Int {
        return 107
    }
}
```

(为什么要用 class 来标识呢？计算型存储属性又如何重写父类呢？)

#### 获取和设置类型属性的值

类型属性通过类型本身访问：

```swift
print(SomeStructure.storedTypeProperty)
// 输出 "Some value."
SomeStructure.storedTypeProperty = "Another value."
print(SomeStructure.storedTypeProperty)
// 输出 "Another value.”
print(SomeEnumeration.computedTypeProperty)
// 输出 "6"
print(SomeClass.computedTypeProperty)
// 输出 "27"
```

## 方法

类、结构体、枚举都可以定义实例、类型方法（这和 oc 中不同，oc 只能在类中定义方法）。

### 实例方法

实例方法的语法和函数完全一致：

```swift
class Counter {
    var count = 0
    func increment() {
        count += 1
    }
    func incrementBy(amount: Int) {
        count += amount
    }
    func reset() {
        count = 0
    }
}
```

和调用属性一样，用点语法调用实例方法：

```swift
let counter = Counter()
// the initial counter value is 0
counter.increment()
// the counter's value is now 1
counter.increment(by: 5)
// the counter's value is now 6
counter.reset()
// the counter's value is now 0
```

#### self 属性

每个实例都有一个隐式的属性叫做 `self`。我们可以在实例方法中，通过 `self` 获取当前实例。

可以改写上面的 `incremnet()` 方法：

```swift
func increment() {
    self.count += 1
}
```

实际使用中，并不太需要 `self`，因为没必要。那么什么时候是必须的呢？

```swift
struct Point {
    var x = 0.0, y = 0.0
    func isToTheRightOf(x: Double) -> Bool {
        return self.x > x
    }
}
let somePoint = Point(x: 4.0, y: 5.0)
if somePoint.isToTheRightOf(x: 1.0) {
    print("This point is to the right of the line where x == 1.0")
}
// Prints "This point is to the right of the line where x == 1.0"
```

比如上面的代码，实例方法的入参名和实例属性相同，这个时候必须要用 `self` 以示区分。









