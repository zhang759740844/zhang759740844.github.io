title: Swift 3.1 语法学习（四）
date: 2017/2/22 14:07:15  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->
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


