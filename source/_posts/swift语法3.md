title: Swift 3.1 语法学习（三）
date: 2017/2/22 14:07:15  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

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

和 oc 不同，这里的枚举值不会被完全的隐式赋值为 0，1，2，3（但是如果你给定了其中一个的值，其他的值可以被隐式地推断出来）。这些枚举成员本身就是完备的值，比如最上面的这些值的类型是已经明确定义好的 `CompassPoint` 类型。

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

> 关联值和原始值不能同时混合使用

#### 原始值的隐式赋值

使用整数或者字符串作为原始值枚举时，不需要显式赋值，Swift 会自动赋值：

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

通过 `.` 语法，可以访问实例的属性。与 oc 不同的是，Swift 允许直接设置**结构体**属性的子属性。反正通过 `.` 什么都能拿到就是了。

### 结构体类型的成员逐一构造器

**所有结构体（特指结构体）都有一个自动生成的*成员逐一构造器***，用于初始化新结构体实例中成员的属性。新实例中各个属性的初始值可以通过属性的名称传递到成员逐一构造器之中：

```swift
let vga = Resolution(width:640, height: 480)
```

与结构体不同，**类实例没有默认的成员逐一构造器**。详细见后面。

### 结构体和枚举是值类型

什么是值类型？就是我们所说的值传递和引用传递中的值传递。指在被赋给一个变量常量或者传递给一个函数的时候，值会被拷贝。

所有的基本类型都是值拷贝这和以前毫无异议。但是在 Swift 中，**字符串、数组和字典也是值拷贝**，这和 oc 中很不相同。这意味着数组和字典中的所有元素都会被拷贝一份。其实是因为**它们底层都是以结构体的形式实现的**。

```swift
let hd = Resolution(width: 1920, height: 1080)
var cinema = hd
```

例如上面的例子，`cinema` 和 `hd` 相同，但其实在内存中是两个不同的对象。修改其中一个的值不会改变另一个实例相应属性的值。**枚举同样**。

> 这里只是数组对象会被创建一个新的，但是数组里的对象都是引用类型。

### 类是引用类型

这点没啥不同的。就是不同常量或者变量指向内存上的相同地址。



### 恒等运算符

Swift 中内建了两个恒等运算符：

- 等价于（`===`）
- 不等价于 （`!==`）

注意这和“等于” `==` 有什么区别呢？

- "等价于"表示两个**类类型**(注意只能用在类类型中)的常量或者变量引用同一个类实例。
- “等于”表示两个实例的值“相等”或“相同”。

也就是说 `===` 为 `true` ，那么两个类实例必然指向同一块内存地址。`==` 为 `true` 则只要类内属性相同即可。（oc 中的 `==` 就是这里的 `===`，比较的是指针地址。

一般对象相等都是比较地址，即 `===`。如果要使用 `==` 必须要实现 `Equatable` 协议。在 `Equatable` 里声明了这个操作符的接口方法:

```swift
protocol Equatable {
    func ==(lhs: Self, rhs: Self) -> Bool
}
```

实现它：

```swift
class MyClass: Equatable {
  let myProperty: String

  init(s: String) {
    myProperty = s
  }
}

func ==(lhs: MyClass, rhs: MyClass) -> Bool {
  return lhs.myProperty == rhs.myProperty
}

let myClass1 = MyClass(s: "Hello")
let myClass2 = MyClass(s: "Hello")
myClass1 == myClass2 // true
myClass1 != myClass2 // false
myClass1 === myClass2 // false
myClass1 !== myClass2 // true
```





### 类和结构体的选择

**结构体实例总是通过值传递，类实例总是通过引用传递。**这意味两者适用不同的任务。

其实大部分数据构造都是用类，而非结构体。只有在数据结构非常简单的时候用结构体。



### 字符串、数组、字典的赋值和复制行为

上面也说过了 Swift 中的 `String`，`Array`和`Dictionary`类型均以结构体的形式实现。这意味着被赋值给新的常量或变量，或者被传入函数或方法中时，它们的值会被拷贝。

Objective-C 中`NSArray`和`NSDictionary`类型均以类的形式实现，而并非结构体。它们在被赋值或者被传入函数或方法时，不会发生值拷贝，而是传递现有实例的引用。

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

> **因此数组字典等如果设置为 `let` ，那么数组字典中的项，值类型无法给内容，引用类型无法改地址**

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

> 这里要强调一点。计算属性的 set 方法在 init 方法中是不会被调用的。如果你在初始化方法中给计算属性赋值了，那么这个计算属性直接就等于这个值，而不是再调用 setter 方法。

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

> 这个来实现 kvo 

### 全局变量和局部变量

全局变量是在函数、方法、闭包或任何类型之外定义的变量。局部变量是在函数、方法或闭包内部定义的变量。

**全局的常量或变量都是延迟计算的**，跟延迟存储属性相似，不同的地方在于，全局的常量或变量不需要标记`lazy`修饰符。
局部范围的常量或变量从不延迟计算。

### 类型属性

实例属性属于一个特定类型的实例，每创建一个实例，实例都拥有属于自己的一套属性值，实例之间的属性相互独立。类似于静态变量或常量。

跟实例的存储型属性不同，必须**给存储型类型属性指定默认值**，因为类型本身没有构造器，也就无法在初始化过程中使用构造器给类型属性赋值。
存储型类型属性是延迟初始化的，它们只有在第一次被访问的时候才会被初始化。即使它们被多个线程同时访问，系统也保证只会对其进行一次初始化，并且不需要对其使用 `lazy` 修饰符。

#### 类型属性语法

在 Swift 中，类型属性是作为类型定义的一部分写在类型最外层的花括号内，因此它的作用范围也就在类型支持的范围内。

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

使用了 `class`，子类才能知道这个计算型属性是可以重写的，重写的时候要加上 `override` 表示：

```swift
class OverridedClass: SomeClass {
    override class var overrideableComputedTypeProperty: Int {
        return 20
    }
}
```

> 其实计算型属性就相当于是个方法。

#### 获取和设置类型属性的值

类型属性通过类型本身访问：

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

**类、结构体、枚举都可以定义实例、类型方法**（这和 oc 中不同，oc 只能在类中定义方法）。

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

#### 实例方法中修改值类型

结构体和枚举是**值类型**。默认情况下，值类型的属性不能在它的实例方法中被修改（也就是说一般创建好之后，结构体和枚举一般就不让修改了）。

如果你想在该实例方法中修改结构体和枚举的属性，那么需要在该方法前加一个可变标记，就可以改变它的值了。这个方法做的任何改变都会在方法执行结束时写回到原始结构中。

要使用`可变`方法，将关键字`mutating` 放到方法的`func`关键字之前就可以了：

```swift
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}
var somePoint = Point(x: 1.0, y: 1.0)
somePoint.moveBy(x: 2.0, y: 3.0)
print("The point is now at (\(somePoint.x), \(somePoint.y))")
// Prints "The point is now at (3.0, 4.0)"
```

注意，不能在结构体类型的常量上调用可变方法，因为其属性不能被改变，即使属性是变量属性。这一点在属性一章中已经讲过了。

#### 可变方法中给 self 赋值

可变方法还能够赋给隐含属性`self`一个全新的实例。上面`Point`的例子可以用下面的方式改写：

```swift
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        self = Point(x: x + deltaX, y: y + deltaY)
    }
}
```

枚举的可变方法可以把`self`设置为同一枚举类型中不同的成员：

```swift
enum TriStateSwitch {
    case off, low, high
    mutating func next() {
        switch self {
        case .off:
            self = .low
        case .low:
            self = .high
        case .high:
            self = .off
        }
    }
}
var ovenLight = TriStateSwitch.low
ovenLight.next()
// ovenLight is now equal to .high
ovenLight.next()
// ovenLight is now equal to .off
```

上面的例子中定义了一个三态开关的枚举。每次调用`next()`方法时，开关在不同的电源状态（`Off`，`Low`，`High`）之间循环切换。

### 类型方法

实例方法是被某个类型的实例调用的方法。你也可以定义在类型本身上调用的方法，这种方法就叫做**类型方法**（Type Methods）。在方法的`func`关键字之前加上关键字`static`，来指定类型方法。**类还可以用关键字`class`来允许子类重写父类的方法实现**。

在 Objective-C 中，你只能为 Objective-C 的类类型（classes）定义类型方法（type-level methods）。在 Swift 中，你可以为所有的类、结构体和枚举定义类型方法。每一个类型方法都被它所支持的类型显式包含。

用点语法调用类型方法：

```swift
class SomeClass {
    class func someTypeMethod() {
        // type method implementation goes here
    }
}
SomeClass.someTypeMethod()
```

类型方法和其他语言的静态方法无二，不多说了。



## 下标

 下标是访问集合列表中元素的跨界方式。可以通过下标索引，设置和获取值，而省略调用相应的存取方法。你可以定义有多个入参的下标满足自定义类型的需求。

### 下标语法

下标允许你通过在实例名称后面的方括号中传入一个或者多个索引值来对实例进行存取。语法类似于实例方法语法和计算型属性语法的混合。与定义实例方法类似，定义下标使用`subscript`关键字，指定一个或多个输入参数和返回类型。下标可以设定为读写或只读。这种行为由 getter 和 setter 实现，有点类似计算型属性：

```swift
subscript(index: Int) -> Int {
    get {
      // 返回一个适当的 Int 类型的值
    }

    set(newValue) {
      // 执行适当的赋值操作
    }
}
```

`newValue`的类型和下标的返回类型相同。如同计算型属性，可以不指定 setter 的参数（`newValue`）。如果不指定参数，setter 会提供一个名为`newValue`的默认参数。

如同只读计算型属性，可以省略只读下标的`get`关键字：

```swift
subscript(index: Int) -> Int {
    // 返回一个适当的 Int 类型的值
}
```

下面看一个例子：

```swift
struct TimesTable {
    let multiplier: Int
    subscript(index: Int) -> Int {
        return multiplier * index
    }
}
let threeTimesTable = TimesTable(multiplier: 3)
print("six times three is \(threeTimesTable[6])")
// 输出 "six times three is 18"
```

### 下标选项

一个类或结构体可以根据自身需要提供多个下标实现，使用下标时将通过入参的数量和类型进行区分，自动匹配合适的下标，这就是*下标的重载*。

虽然接受单一入参的下标是最常见的，但也可以根据情况定义接受多个入参的下标。例如下例定义了一个`Matrix`结构体，用于表示一个`Double`类型的二维矩阵。`Matrix`结构体的下标接受两个整型参数：

```swift
struct Matrix {
    let rows: Int, columns: Int
    var grid: [Double]
    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        grid = Array(count: rows * columns, repeatedValue: 0.0)
    }
    func indexIsValidForRow(row: Int, column: Int) -> Bool {
        return row >= 0 && row < rows && column >= 0 && column < columns
    }
    subscript(row: Int, column: Int) -> Double {
        get {
            assert(indexIsValidForRow(row, column: column), "Index out of range")
            return grid[(row * columns) + column]
        }
        set {
            assert(indexIsValidForRow(row, column: column), "Index out of range")
            grid[(row * columns) + column] = newValue
        }
    }
}
```

使用：

```swift
var matrix = Matrix(rows: 2, columns: 2)
matrix[0, 1] = 1.5
matrix[1, 0] = 3.2
```

其中使用了断言，断言在下标越界时触发。


