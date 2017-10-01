title: Swift 3.1 语法学习（二）
date: 2017/2/22 14:07:14  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->
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

> 一定要注意，swift 中的 switch 一定要有 default，一定要有 default，一定要有 default。

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

> 这个就相当于是或操作，下面还有与操作

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

> 这就引导我们，没有执行方法的条件，就不要写

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

> 这种方式可以一定程度上替代一些判断数值的 if…else

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

> 元组其实相当于 if…else… 中 && 的操作

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

> 这种值绑定的方式，可以把函数中要用到的变量先确定下来。

> 最后一定要有一个 `let(x,y)`，表示一个 default 操作，否则编译器会抛出异常。

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

> 这里的 where 不仅适用于元组，也可以是普通的 `case let x where x==1:`但是如果不用元组就没有 if…else… 简洁。

> switch 能在一定程度上代替判断相等，以及判断在某一个范围的简单 if…else 操作 

### 控制转移声明

控制转移声明包括：

- continue
- break
- fallthrough
- return
- throw

除了 `fallthrough` 其他都差不多。这里要注意一下在 switch 中使用的 `break`。由于 switch 中的 case 里的执行语句不能为空，所以**如果匹配到一种情况不需要操作，可以直接用 `break`:**

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

> 适用于那种满足了某个条件包含其他条件的操作，即要执行 A 那么 B 也要执行。最上面的永远是执行的最多的。

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

严格来说，其实无返回类型的函数还是返回了一个值，即使没有返回值定义。函数没有定义返回类型但返 回了一个 `void` 返回类型的特殊值。它是一个空的元组，可以写为`return ()`

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

如果返回的元组可能没有值，那么可以使用可选的元组作为返回类型，来表示整个元组可以为 `nil`。**在元组之后加上 `?` 来表示可选类型，例如 `(Int,Int)?`** （注意不要写成 `(Int?,Int?)` 这个表示元组中的元素是可选的）。

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

注意，参数顺序还是不能错的，否则产生异常。

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
5. 还有一种情形：`func arithmeticMean(_ numbers: Double...,_ anotherNumber: Double)`，这种情况下由于两个入参都是缺省的，所有传入的参数都被 `numbers` 接收，第二个参数无法接收参数。最好不要这样写。

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

> 可以直接写成 `mathFunction = addTwoInts` 通过类型推断决定类型

现在就可以像使用 `addTwoInts` 方法一样使用 `mathFunction` 了：

```swift
print("Result: \(mathFunction(2, 3))")
// Prints "Result: 5"
```

> **注意，使用函数类型的地方就不能再有 argument label 以及 parameter name 了**

就算 addTwoInts 的两个入参是有 argument label 的，这里调用 mathFunction 的时候也要省略。mathFunction 不能自己定义 argument label 以及 parameter name ，只能赋予函数之后直接使用。例子：

![添加桥接文件](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/swift_example.png?raw=true) 

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

> 由于传递函数不能改变 argument label 以及 parameter name，所以使用类型推断更方便一些，不要再写一遍函数类型了。

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

单行表达式闭包可以通过**隐藏 `return`** 关键字来隐式返回单行表达式的结果，如上版本的例子可以改写为：

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

// 使用尾部闭包如果没有参数，可以省略括号
someFunctionThatTakesAclosure {
  
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

### 捕获值

**闭包可以在捕获其所定义的上下文中的变量和常量**。即使定义这些常量和变量的原作用域已经不存在，闭包仍然可以在闭包函数体内引用和修改这些值。

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

另外，看到 `completionHandlers.first?()` 了么，虽然这列数组是非可选的，但是这个 `first` 方法返回的是一个可选类型，所以使用的时候要加上 `?` 或者 `!`

> 类比 oc 中block，在某个方法中传递回调 block。这个 block 使用的外部变量都要设为 weak，以防引用循环。同样，这里调用的时候要用 self，外加 @escaping 标记，来告诉编译器，要优化这里的存储，以防引用循环。

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

> 貌似只针对一句话的闭包