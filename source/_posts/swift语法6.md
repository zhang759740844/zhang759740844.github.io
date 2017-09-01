title: Swift 3.1 语法学习（六）
date: 2017/2/22 14:07:19  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

## 拓展

*扩展* 就是为一个已有的类、结构体、枚举类型或者协议类型添加新功能。这包括在没有权限获取原始源代码的情况下扩展类型的能力。

> 类似于 oc 的 category

Swift 中的扩展可以：

- 添加**计算型属性和计算型类型属性**
- 定义实例方法和类型方法
- 提供新的构造器(具体看下面)
- 定义下标
- 定义和使用新的嵌套类型
- 使一个已有类型符合某个协议

> 拓展可以为一个类型添加新的功能，但是不能重写已有功能，也不可以添加存储型的属性类属性，这和 oc 一致
>

### 拓展语法

使用关键字 `extension` 来声明扩展：

```swift
extension SomeType {
    // 为 SomeType 添加的新功能写到这里
}
```

可以通过扩展来扩展一个已有类型，使其采纳一个或多个协议。在这种情况下，无论是类还是结构体，协议名字的书写方式完全一样：

```swift
extension SomeType: SomeProtocol, AnotherProctocol {
    // 协议实现写到这里
}
```

> 如果你通过扩展为一个已有类型添加新功能，那么新功能对该类型的所有已有实例都是可用的，即使它们是在这个扩展定义之前创建的。

### 计算型属性

扩展可以为已有类型添加计算型实例属性和计算型类型属性。

```swift
extension Double {
    var km: Double { return self * 1_000.0 }
    var m : Double { return self }
    var cm: Double { return self / 100.0 }
    var mm: Double { return self / 1_000.0 }
    var ft: Double { return self / 3.28084 }
}
let oneInch = 25.4.mm
print("One inch is \(oneInch) meters")
// 打印 “One inch is 0.0254 meters”
let threeFeet = 3.ft
print("Three feet is \(threeFeet) meters")
// 打印 “Three feet is 0.914399970739201 meters”
```

由于拓展里不会有非可选属性，所以，这里可以直接获取 self

> 扩展可以添加新的计算型属性，但是不可以添加存储型属性，也不可以为已有属性添加属性观察器。

### 构造器

扩展能为**类**添加新的便利构造器，但是它们**不能为类添加新的指定构造器或析构器**。指定构造器和析构器必须总是由原始的类实现来提供。

> 复习一下，便利构造器和指定构造器是类中的两种不同类型构造器，不是结构体中的。因此，上面的话不适用于结构体，结构体中还是可以用拓展添加构造器的。

```swift
struct Size {
    var width = 0.0, height = 0.0
}
struct Point {
    var x = 0.0, y = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
}

let defaultRect = Rect()
let memberwiseRect = Rect(origin: Point(x: 2.0, y: 2.0),
    size: Size(width: 5.0, height: 5.0))
    
extension Rect {
    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}

let centerRect = Rect(center: Point(x: 4.0, y: 4.0),
    size: Size(width: 3.0, height: 3.0))
// centerRect 的原点是 (2.5, 2.5)，大小是 (3.0, 3.0)
```

### 方法

扩展可以为已有类型添加新的实例方法和类型方法。下面的例子为 `Int` 类型添加了一个名为 `repetitions` 的实例方法：

```swift
extension Int {
    func repetitions(task: () -> Void) {
        for _ in 0..<self {
            task()
        }
    }
}

3.repetitions(task: {
    print("Hello!")
})
// Hello!
// Hello!
// Hello!
```

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          

### 可变实例方法

通过扩展添加的实例方法也可以修改该实例本身。结构体和枚举类型中修改 `self`或其属性的方法必须将该实例方法标注为 `mutating`，正如来自原始实现的可变方法一样。

```swift
extension Int {
    mutating func square() {
        self = self * self
    }
}
var someInt = 3
someInt.square()
// someInt 的值现在是 9
```

> 所以说嘛 Int 是结构体

### 下标

扩展可以为已有类型添加新下标。这个例子为 Swift 内建类型 `Int` 添加了一个整型下标。该下标 `[n]` 返回十进制数字从右向左数的第 `n` 个数字：

- `123456789[0]` 返回 `9`
- `123456789[1]` 返回 `8`

```swift
extension Int {
    subscript(digitIndex: Int) -> Int {
        var decimalBase = 1
        for _ in 0..<digitIndex {
            decimalBase *= 10
        }
        return (self / decimalBase) % 10
    }
}
746381295[0]
// 返回 5
746381295[1]
// 返回 9
746381295[2]
// 返回 2
746381295[8]
// 返回 7
```

> `0..<digitIndex ` 表示一个范围

### 嵌套类型

扩展可以为已有的类、结构体和枚举添加新的嵌套类型：

```swift
extension Int {
    enum Kind {
        case Negative, Zero, Positive
    }
    var kind: Kind {
        switch self {
        case 0:
            return .Zero
        case let x where x > 0:
            return .Positive
        default:
            return .Negative
        }
    }
}
```

该例子为 `Int` 添加了嵌套枚举。这个名为 `Kind` 的枚举表示特定整数的类型。具体来说，就是表示整数是正数、零或者负数。

> 不能添加属性，但是可以添加枚举

使用：

```swift
func printIntegerKinds(numbers: [Int]) {
    for number in numbers {
        switch number.kind {
        case .Negative:
            print("- ", terminator: "")
        case .Zero:
            print("0 ", terminator: "")
        case .Positive:
            print("+ ", terminator: "")
        }
    }
    print("")
}
printIntegerKinds([3, 19, -27, 0, -6, 0, 7])
// 打印 “+ + - 0 - 0 + ”
```

> 由于已知 `number.kind` 是 `Int.Kind` 类型，因此在 `switch` 语句中，`Int.Kind` 中的所有成员值都可以使用简写形式，例如使用 `.Negative`而不是 `Int.Kind.Negative`。

## 协议

规定了用来实现某一特定任务或者功能的方法、属性，以及其他需要的东西。**类、结构体或枚举**都可以采纳协议，并为协议定义的这些要求提供具体实现。

### 协议语法

```swift
protocol SomeProtocol {
    // 这里是协议的定义部分
}
```

要让自定义类型采纳某个协议，在定义类型时，需要在类型名称后加上协议名称，中间以冒号（`:`）分隔。采纳多个协议时，各协议之间用逗号（`,`）分隔，**拥有父类的类在采纳协议时，应该将父类名放在协议名之前，以逗号分隔**：

```swift
class SomeClass: SomeSuperClass, FirstProtocol, AnotherProtocol {
    // 这里是类的定义部分
}
```

###  属性要求

协议可以要求采纳协议的类型提供特定名称和类型的实例属性或类型属性。协议不指定属性是存储型属性还是计算型属性，它只指定属性的名称和类型。此外，协议还指定属性是可读的还是可读可写的。

如果协议要求属性是可读可写的，那么该属性不能是常量属性或只读的计算型属性。如果协议只要求属性是可读的，那么该属性不仅可以是可读的，如果代码需要的话，还可以是可写的。

协议总是用 `var` 关键字来声明变量属性，在类型声明后加上 `{ set get }` 来表示属性是可读可写的，只读属性则用 `{ get }` 来表示：

```swift
protocol SomeProtocol {
    var mustBeSettable: Int { get set }
    var doesNotNeedToBeSettable: Int { get }
}
```

在协议中定义类型属性时，总是使用 `static` 关键字作为前缀。

```swift
protocol AnotherProtocol {
    static var someTypeProperty: Int { get set }
}
```

> 当类类型采纳协议时，除了 `static` 关键字，还可以使用 `class` 关键字来声明类型属性

 ### 方法要求

这些方法作为协议的一部分，像普通方法一样放在协议的定义中，但是不需要大括号和方法体。可以在协议中定义具有可变参数的方法，和普通方法的定义方式相同。但是，**不支持为协议中的方法的参数提供默认值**。

正如属性要求中所述，在协议中定义类方法的时候，总是使用 `static` 关键字作为前缀。

```swift
protocol SomeProtocol {
    static func someTypeMethod()
}
```

> 当类类型采纳协议时，除了 `static` 关键字，还可以使用 `class` 关键字作为前缀

```swift
protocol RandomNumberGenerator {
    func random() -> Double
}

class LinearCongruentialGenerator: RandomNumberGenerator {
    var lastRandom = 42.0
    let m = 139968.0
    let a = 3877.0
    let c = 29573.0
    func random() -> Double {
        lastRandom = ((lastRandom * a + c) % m)
        return lastRandom / m
    }
}
let generator = LinearCongruentialGenerator()
print("Here's a random number: \(generator.random())")
// 打印 “Here's a random number: 0.37464991998171”
print("And another one: \(generator.random())")
// 打印 “And another one: 0.729023776863283”
```

### Mutating 方法要求

如果你在协议中定义了一个实例方法，该方法会改变采纳该协议的类型的实例，那么在定义协议时需要在方法前加 `mutating` 关键字。这使得结构体和枚举能够采纳此协议并满足此方法要求。

> 实现协议中的 `mutating` 方法时，若是类类型，则不用写 `mutating` 关键字。而对于结构体和枚举，则必须写 `mutating` 关键字。

```swift
protocol Togglable {
    mutating func toggle()
}

enum OnOffSwitch: Togglable {
    case Off, On
    mutating func toggle() {
        switch self {
        case Off:
            self = On
        case On:
            self = Off
        }
    }
}
var lightSwitch = OnOffSwitch.Off
lightSwitch.toggle()
// lightSwitch 现在的值为 .On
```

### 构造器要求

你可以像编写普通构造器那样，在协议的定义里写下构造器的声明，但不需要写花括号和构造器的实体：

```swift
protocol SomeProtocol {
    init(someParameter: Int)
}
```

#### 构造器要求在类中的实现

你可以在采纳协议的类中实现构造器，**无论是作为指定构造器，还是作为便利构造器**。无论哪种情况，你都必须为构造器实现标上 `required` 修饰符：

```swift
class SomeClass: SomeProtocol {
    required init(someParameter: Int) {
        // 这里是构造器的实现部分
    }
}
```

使用 `required` 修饰符可以确保所有子类也必须提供此构造器实现，从而也能符合协议。

> 如果类已经被标记为 `final`，那么不需要在协议构造器的实现中使用 `required` 修饰符，因为 `final` 类不能有子类。

如果一个子类重写了父类的指定构造器，并且该构造器满足了某个协议的要求，那么该构造器的实现需要同时标注 `required` 和 `override` 修饰符：

```swift
protocol SomeProtocol {
    init()
}

class SomeSuperClass {
    init() {
        // 这里是构造器的实现部分
    }
}

class SomeSubClass: SomeSuperClass, SomeProtocol {
    // 因为采纳协议，需要加上 required
    // 因为继承自父类，需要加上 override
    required override init() {
        // 这里是构造器的实现部分
    }
}
```

> 貌似也没那么巧，重写的方法和继承的方法同名吧

#### 可失败构造器要求

采纳协议的类型可以通过可失败构造器（`init?`）或非可失败构造器（`init`）来满足协议中定义的可失败构造器要求。协议中定义的非可失败构造器要求可以通过非可失败构造器（`init`）或隐式解包可失败构造器（`init!`）来满足。

### 协议作为类型

尽管协议本身并未实现任何功能，但是协议可以被当做一个成熟的类型来使用。

> 注意，Swift 中协议也是一种类型。

```swift
class Dice {
    let sides: Int
    let generator: RandomNumberGenerator
    init(sides: Int, generator: RandomNumberGenerator) {
        self.sides = sides
        self.generator = generator
    }
    func roll() -> Int {
        return Int(generator.random() * Double(sides)) + 1
    }
}

//LinearCongruentialGenerator 和 RandomNumberGenerator 上面定义过
var d6 = Dice(sides: 6, generator: LinearCongruentialGenerator())
for _ in 1...5 {
    print("Random dice roll is \(d6.roll())")
}
// Random dice roll is 3
// Random dice roll is 5
// Random dice roll is 4
// Random dice roll is 5
// Random dice roll is 4
```

`RandomNumberGenerator` 是上面定义的协议名

> oc 中使用某个协议是 `id<协议名>`。swift 中直接将协议当成一种类型了

### 通过扩展添加协议一致性

即便无法修改源代码，依然可以通过扩展令已有类型采纳并符合协议。

> 通过扩展令已有类型采纳并符合协议时，该类型的所有实例也会随之获得协议中定义的各项功能。

```swift
protocol TextRepresentable {
    var textualDescription: String { get }
}

extension Dice: TextRepresentable {
    var textualDescription: String {
        return "A \(sides)-sided dice"
    }
}
```

### 协议的继承

协议能够继承一个或多个其他协议，可以在继承的协议的基础上增加新的要求。协议的继承语法与类的继承相似，多个被继承的协议间用逗号分隔：

```swift
protocol PrettyTextRepresentable: TextRepresentable {
    var prettyTextualDescription: String { get }
}
```

任何采纳 `PrettyTextRepresentable` 协议的类型在满足该协议的要求时，也必须满足 `TextRepresentable` 协议的要求。

### 类类型专属协议

你可以在协议的继承列表中，通过添加 `class` 关键字来限制协议只能被类类型采纳，而结构体或枚举不能采纳该协议。`class` 关键字必须第一个出现在协议的继承列表中，在其他继承的协议之前：

```swift
protocol SomeClassOnlyProtocol: class, SomeInheritedProtocol {
    // 这里是类类型专属协议的定义部分
}
```

### 协议合成

有时候需要同时采纳多个协议，你可以将多个协议采用 `SomeProtocol & AnotherProtocol` 这样的格式进行组合，称为 *协议合成*。你可以罗列任意多个你想要采纳的协议，以与符号(`&`)分隔。

```swift
protocol Named {
    var name: String { get }
}
protocol Aged {
    var age: Int { get }
}
struct Person: Named, Aged {
    var name: String
    var age: Int
}
func wishHappyBirthday(to celebrator: Named & Aged) {
    print("Happy birthday, \(celebrator.name), you're \(celebrator.age)!")
}
let birthdayPerson = Person(name: "Malcolm", age: 21)
wishHappyBirthday(to: birthdayPerson)
// 打印 “Happy birthday Malcolm - you're 21!”
```

> 协议合成并不会生成新的、永久的协议类型，而是将多个协议中的要求合成到一个只在局部作用域有效的临时协议中。

### 检查协议一致性

前面说过协议也能作为类型使用。那么也就可以使用[类型转换](http://www.swift51.com/swift3.0/chapter2/19_Type_Casting.html)中描述的 `is` 和 `as` 操作符来检查协议一致性，即是否符合某协议，并且可以转换到指定的协议类型。

```swift
for object in objects {
    if let objectWithArea = object as? HasArea {
        print("Area is \(objectWithArea.area)")
    } else {
        print("Something that doesn't have an area")
    }
}
// Area is 12.5663708
// Area is 243610.0
// Something that doesn't have an area
```

### 可选的协议要求

协议可以定义可选要求，采纳协议的类型可以选择是否实现这些要求。在协议中使用 `optional` 关键字作为前缀来定义可选要求。使用可选要求时（例如，可选的方法或者属性），它们的类型会自动变成可选的。比如，一个类型为 `(Int) -> String`的方法会变成 `((Int) -> String)?`。需要注意的是整个函数类型是可选的，而不是函数的返回值。

协议中的可选要求可通过可选链式调用来使用，因为采纳协议的类型可能没有实现这些可选要求。类似 `someOptionalMethod?(someArgument)` 这样，你可以在可选方法名称后加上 `?` 来调用可选方法。

```swift
@objc protocol CounterDataSource {
    @objc optional func increment(forCount count: Int) -> Int
    @objc optional var fixedIncrement: Int { get }
}
```

一个完整的例子：

```swift
class Counter {
    var count = 0
    var dataSource: CounterDataSource?
    func increment() {
        if let amount = dataSource?.increment?(forCount: count) {
            count += amount
        } else if let amount = dataSource?.fixedIncrement {
            count += amount
        }
    }
}

class ThreeSource: NSObject, CounterDataSource {
    let fixedIncrement = 3
}

var counter = Counter()
counter.dataSource = ThreeSource()
for _ in 1...4 {
    counter.increment()
    print(counter.count)
}
// 3
// 6
// 9
// 12
```

> 注意 `fixedIncrement` 在协议中是 var，但是实现是可以设置为 `let`

### 协议扩展

协议可以通过扩展来为采纳协议的类型提供属性、方法以及下标的实现。通过这种方式，**你可以基于协议本身来实现这些功能，而无需在每个采纳协议的类型中都重复同样的实现，也无需使用全局函数。**

例如，可以扩展 `RandomNumberGenerator` 协议来提供 `randomBool()` 方法：

```swift
extension RandomNumberGenerator {
    func randomBool() -> Bool {
        return random() > 0.5
    }
}
```

通过协议扩展，所有采纳协议的类型，都能自动获得这个扩展所增加的方法实现，无需任何额外修改。

> 这个协议拓展的本意是为了给协议提供默认实现，这样就可以不必在每个的类中都实现该协议的方法。但是，这样一搞就有点像多继承了。多继承的问题是如果继承了多个实现相同方法的协议，那么调用的时候该如何取舍？
>
> Swift 中的类如果拥有两个有相同方法名的协议拓展，那么编译都不能通过，除非在类中自己实现了。

#### 提供默认实现

可以通过协议扩展来为协议要求的属性、方法以及下标提供默认的实现。如果采纳协议的类型为这些要求提供了自己的实现，那么这些自定义实现将会替代扩展中的默认实现被使用。

如上面的 `PrettyTextRepresentable` 协议的方法：

```swift
extension PrettyTextRepresentable  {
    var prettyTextualDescription: String {
        return textualDescription
    }
}
```



## 泛型

### 泛型函数

泛型的使用实例：

```swift
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
    let temporaryA = a
    a = b
    b = temporaryA
}

var someInt = 3
var anotherInt = 107
swapTwoValues(&someInt, &anotherInt)
// someInt is now 107, and anotherInt is now 3

var someString = "hello"
var anotherString = "world"
swapTwoValues(&someString, &anotherString)
```

例子中，当函数第一次被调用时，`T` 被 `Int` 替换，第二次调用时，被 `String` 替换。可提供多个类型参数，将它们都写在尖括号中，用逗号分开。

> 这里范型名为 T，可以替换为任意字符

### 泛型类型

除了泛型函数，Swift 还允许你定义泛型类型。这些自定义类、结构体和枚举可以适用于任何类型，类似于 `Array` 和 `Dictionary`。

```swift
struct Stack<Element> {
    var items = [Element]()
    mutating func push(item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }
}

var stackOfStrings = Stack<String>()
```

### 拓展一个泛型类型

当你扩展一个泛型类型的时候，你并不需要在扩展的定义中提供类型参数列表。原始类型定义中声明的类型参数列表在扩展中可以直接使用，并且这些来自原始类型中的参数名称会被用作原始定义中类型参数的引用。

```swift
extension Stack {
    var topItem: Element? {
        return items.isEmpty ? nil : items[items.count - 1]
    }
}
```



### 类型约束

上面的泛型是所有类型都可用的，但是实际使用中，可能需要排除掉一些类型。比如 `Dictionary` 类型对字典的键做了限制，只有可以 hash 的类型才能成为键。

当你创建自定义泛型类型时，你可以定义你自己的类型约束

#### 类型约束语法

可以在一个类型参数名后面放置一个类名或者协议名，并用冒号进行分隔，来定义类型约束，它们将成为类型参数列表的一部分。

```swift
func someFunction<T: SomeClass, U: SomeProtocol>(someT: T, someU: U) {
    // 这里是泛型函数的函数体部分
}
```

上面这个函数有两个类型参数。第一个类型参数 `T`，有一个要求 `T` 必须是 `SomeClass` 子类的类型约束；第二个类型参数 `U`，有一个要求 `U` 必须符合 `SomeProtocol` 协议的类型约束。



### 关联类型

为协议提供泛型即使用关联类型。关联类型可以通过 `associatedtype` 关键字来指定。

#### 关联类型实例

下面定义了一个关联类型 `ItemType`：

```swift
protocol Container {
    associatedtype ItemType
    mutating func append(item: ItemType)
    var count: Int { get }
    subscript(i: Int) -> ItemType { get }
}
```

`Container` 协议声明了一个关联类型 `ItemType`，写作 `associatedtype ItemType`。这个协议无法定义 `ItemType` 是什么类型的别名，这个信息将留给遵从协议的类型来提供。

使用：

```swift
struct IntStack: Container {
    // IntStack 的原始实现部分
    var items = [Int]()
    mutating func push(item: Int) {
        items.append(item)
    }
    mutating func pop() -> Int {
        return items.removeLast()
    }
    // Container 协议的实现部分
    typealias ItemType = Int
    mutating func append(item: Int) {
        self.push(item)
    }
    var count: Int {
        return items.count
    }
    subscript(i: Int) -> Int {
        return items[i]
    }
}
```

这里显式地指定 `ItemType` 为 `Int` 类型，即 `typealias ItemType = Int`，从而将 `Container` 协议中抽象的 `ItemType` 类型转换为具体的 `Int` 类型。

> typealias 总算有用了

不过由于类型推断，可以不用声明 `ItemType` 为 `Int`：

```swift
struct Stack<Element>: Container {
    // Stack<Element> 的原始实现部分
    var items = [Element]()
    mutating func push(item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }
    // Container 协议的实现部分
    mutating func append(item: Element) {
        self.push(item)
    }
    var count: Int {
        return items.count
    }
    subscript(i: Int) -> Element {
        return items[i]
    }
}
```

这一次，占位类型参数 `Element` 被用作 `append(_:)` 方法的 `item` 参数和下标的返回类型。Swift 可以据此推断出 `Element` 的类型即是 `ItemType` 的类型。





