title: Swift 3.1 语法学习（四）
date: 2017/2/22 14:07:17  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

## 继承

只有类可以被继承，结构体和枚举不能被继承

### 定义一个基类

不继承于其它类的类，称之为*基类*。

```swift
class Vehicle {
    var currentSpeed = 0.0
    var description: String {
        return "traveling at \(currentSpeed) miles per hour"
    }
    func makeNoise() {
        // 什么也不做-因为车辆不一定会有噪音
    }
}
```

### 子类生成

为了指明某个类的超类，将超类名写在子类名的后面，用冒号分隔：

```swift
class SomeClass: SomeSuperclass {
    // 这里是子类的定义
}
```

继承上面的基类：

```swift
class Bicycle: Vehicle {
    var hasBasket = false
}
```

除了继承了 `Vehicle` 的属性外，还定义了一个 `hasBasket` 属性。

基本和其他语言的继承无异。

### 重写

如果要重写某个特性，你需要在重写定义的前面加上`override`关键字。这么做，你就表明了你是想提供一个重写版本，而非错误地提供了一个相同的定义。任何缺少`override`关键字的重写都会在编译时被诊断为错误。

`override`关键字会提醒 Swift 编译器去检查该类的超类（或其中一个父类）是否有匹配重写版本的声明。

#### 访问超类的方法属性和下标

使用 `super` 关键字。

#### 重写属性

你可以重写继承来的实例属性或类型属性，在属性前加上 `override` 关键字，提供自己定制的 getter 和 setter，或添加属性观察器使重写的属性可以观察属性值什么时候发生改变。

##### 重写属性的 Getters 和 Setters

你可以提供定制的 getter（或 setter）来重写任意继承来的属性，无论继承来的属性是存储型的还是计算型的属性。子类并不知道继承来的属性是存储型的还是计算型的，它只知道继承来的属性会有一个名字和类型。**你在重写一个属性时，必需将它的名字和类型都写出来。**这样才能使编译器去检查你重写的属性是与超类中同名同类型的属性相匹配的。

你可以将一个继承来的只读属性重写为一个读写属性，只需要在重写版本的属性里提供 getter 和 setter 即可。但是，你不可以将一个继承来的读写属性重写为一个只读属性。

如果你不想在重写版本中的 getter 里修改继承来的属性值，你可以直接通过`super.someProperty`来返回继承来的值，其中`someProperty`是你要重写的属性的名字。

##### 重写属性观察器

你可以通过重写属性为一个继承来的属性添加属性观察器。这样一来，当继承来的属性值发生改变时，你就会被通知到，无论那个属性原本是如何实现的。

```swift
class AutomaticCar: Car {
    override var currentSpeed: Double {
        didSet {
            gear = Int(currentSpeed / 10.0) + 1
        }
    }
}

let automatic = AutomaticCar()
automatic.currentSpeed = 35.0
print("AutomaticCar: \(automatic.description)")
// AutomaticCar: traveling at 35.0 miles per hour in gear 4
```

> 重写的是属性的方法，重写不能改变非计算型属性原有的值。所以，比如上面的 currentSpeed 不能赋值，因为父类 Car 里已经有这个属性的初值了。

### 防止重写

可以通过把方法，属性或下标标记为*final*来防止它们被重写，只需要在声明关键字前加上`final`修饰符即可（例如：`final var`，`final func`，`final class func`，以及`final subscript`）。

你可以通过在关键字`class`前添加`final`修饰符（`final class`）来将整个类标记为 final 的。这样的类是不可被继承的，试图继承这样的类会导致编译报错。

## 构造过程

构造过程是使用类、结构体或枚举类型的实例之前的准备过程。在新实例可用前必须执行这个过程，具体操作包括设置实例中每个存储型属性的初始值和执行其他必须的设置或初始化工作。

通过定义构造器（`Initializers`）来实现构造过程，这些构造器可以看做是用来创建特定类型新实例的特殊方法。与 Objective-C 中的构造器不同，Swift 的构造器无需返回值。

类的实例也可以通过定义析构器（`deinitializer`）在实例释放之前执行特定的清除工作。

### 存储属性初始赋值

类和结构体在创建实例时，必须为所有存储型属性设置合适的初始值。存储型属性的值不能处于一个未知的状态。你可以在构造器中为存储型属性赋初值，也可以在定义属性时为其设置默认值。

注意，当你为存储型属性设置默认值或者在构造器中为其赋值时，它们的值是被直接设置的，**不会触发任何属性观察者**（`property observers`）。

#### 构造器

构造器在创建某个特定类型的新实例时被调用。它的最简形式类似于一个不带任何参数的实例方法，以关键字`init`命名：

```swift
init() {
    // 在此处执行构造过程
}
```

例子：

```swift
struct Fahrenheit {
    var temperature: Double
    init() {
        temperature = 32.0
    }
}
var f = Fahrenheit()
print("The default temperature is \(f.temperature)° Fahrenheit")
// 输出 "The default temperature is 32.0° Fahrenheit”
```

#### 默认属性值

你可以使用更简单的方式在定义结构体`Fahrenheit`时为属性`temperature`设置默认值：

```swift
struct Fahrenheit {
    var temperature = 32.0
}
```

### 自定义构造过程

你可以通过输入参数和可选类型的属性来自定义构造过程，也可以在构造过程中修改常量属性。

#### 构造参数

自定义`构造过程`时，可以在定义中提供构造参数，指定所需值的类型和名字。构造参数的功能和语法跟函数和方法的参数相同。

```swift
struct Celsius {
    var temperatureInCelsius: Double
    init(fromFahrenheit fahrenheit: Double) {
        temperatureInCelsius = (fahrenheit - 32.0) / 1.8
    }
    init(fromKelvin kelvin: Double) {
        temperatureInCelsius = kelvin - 273.15
    }
}
let boilingPointOfWater = Celsius(fromFahrenheit: 212.0)
// boilingPointOfWater.temperatureInCelsius 是 100.0
let freezingPointOfWater = Celsius(fromKelvin: 273.15)
// freezingPointOfWater.temperatureInCelsius 是 0.0”
```

第一个构造器拥有一个构造参数，其外部名字为`fromFahrenheit`，内部名字为`fahrenheit`；第二个构造器也拥有一个构造参数，其外部名字为`fromKelvin`，内部名字为`kelvin`。这两个构造器都将唯一的参数值转换成摄氏温度值，并保存在属性`temperatureInCelsius`中。

和函数和方法的定义类似

#### 可选属性类型

如果你定制的类型包含一个逻辑上允许取值为空的存储型属性——无论是因为它无法在初始化时赋值，还是因为它在之后某个时间点可以赋值为空——你都需要将它定义为`可选类型`（optional type）。可选类型的属性将自动初始化为`nil`，表示这个属性是有意在初始化时设置为空的。

例子：

```swift
class SurveyQuestion {
    var text: String
    var response: String?
    init(text: String) {
        self.text = text
    }
    func ask() {
        print(text)
    }
}
let cheeseQuestion = SurveyQuestion(text: "Do you like cheese?")
cheeseQuestion.ask()
// Prints "Do you like cheese?"
cheeseQuestion.response = "Yes, I do like cheese."
```

#### 构造过程中常量属性的赋值

你可以在构造过程中的任意时间点给常量属性指定一个值，只要在构造过程结束时是一个确定的值。一旦常量属性被赋值，它将永远不可更改。

**对于类的实例来说，它的常量属性只能在定义它的类的构造过程中赋值；不能在子类的构造过程中赋值。**道理很浅显，由于常量不论是否可选都要被赋值，不能通过类型推断，所以父类初始化的方法中必须要对常量赋值。又因为常量在赋值之后就不能更改了，所以父类中赋值之后，就不能再在子类中赋值了。

可以修改上面的`SurveyQuestion`示例，用常量属性替代变量属性`text`，表示问题内容`text`在`SurveyQuestion`的实例被创建之后不会再被修改。尽管`text`属性现在是常量，我们仍然可以在类的构造器中设置它的值：

```swift
class SurveyQuestion {
    let text: String
    var response: String?
    init(text: String) {
        self.text = text
    }
    func ask() {
        print(text)
    }
}
let beetsQuestion = SurveyQuestion(text: "How about beets?")
beetsQuestion.ask()
// 输出 "How about beets?"
beetsQuestion.response = "I also like beets. (But not with cheese.)"
```

### 默认构造器

**如果结构体或类的所有属性都有默认值，同时没有自定义的构造器，那么 Swift 会给这些结构体或类提供一个默认构造器（default initializers）**。这个默认构造器将简单地创建一个所有属性值都设置为默认值的实例。

下面例子中创建了一个类`ShoppingListItem`，它封装了购物清单中的某一物品的属性：名字（`name`）、数量（`quantity`）和购买状态 `purchase state`：

```swift
class ShoppingListItem {
    var name: String?
    var quantity = 1
    var purchased = false
}
var item = ShoppingListItem()
```

由于`ShoppingListItem`类中的所有属性都有默认值，且它是没有父类的基类，它将自动获得一个可以为所有属性设置默认值的默认构造器（尽管代码中没有显式为`name`属性设置默认值，但由于`name`是可选字符串类型，它将默认设置为`nil`）。上面例子中使用默认构造器创造了一个`ShoppingListItem`类的实例（使用`ShoppingListItem()`形式的构造器语法），并将其赋值给变量`item`。

> 默认构造器要求所有属性都有默认值。所以上面例子里的 `name` 必须是可选的。如果是非可选的就会产生异常，你需要添加一个 init 方法来初始化这个 非可选`name`



#### 结构体的逐一成员构造器

除了上面提到的默认构造器，**如果结构体没有提供自定义的构造器，它们将自动获得一个逐一成员构造器，即使结构体的存储型属性没有默认值**。

逐一成员构造器是用来初始化结构体新实例里成员属性的快捷方法。我们在调用逐一成员构造器时，通过与成员属性名相同的参数名进行传值来完成对成员属性的初始赋值。

```swift
struct Size {
    var width = 0.0, height = 0.0
}
let twoByTwo = Size(width: 2.0, height: 2.0)
```

(类就不存在这种逐一成员构造器)

### 值类型的构造器代理

构造器可以通过调用其它构造器来完成实例的部分构造过程。这一过程称为构造器代理，它能减少多个构造器间的代码重复。

构造器代理的实现规则和形式在值类型和类类型中有所不同。**值类型（结构体和枚举类型）不支持继承**，所以构造器代理的过程相对简单，因为它们只能代理给自己的其它构造器。

**如果你为某个值类型定义了一个自定义的构造器，你将无法访问到默认构造器（如果是结构体，还将无法访问逐一成员构造器）。**这种限制可以防止你为值类型增加了一个额外的且十分复杂的构造器之后,仍然有人错误的使用自动生成的构造器。

举例：

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
    init() {}
    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }
    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}
```

> 就是类中的构造器调用类中的另一个构造器

### 类的继承和构造过程

**类里面的所有存储型属性——包括所有继承自父类的属性——都必须在构造过程中设置初始值。**（基本上所有后面的所有的限制都是围绕这一规定）

Swift 为类类型提供了两种构造器来确保实例中所有存储型属性都能获得初始值，它们分别是指定构造器和便利构造器。

#### 指定构造器和便利构造器

##### 指定构造器

*指定构造器*（designated initializers）是类中最主要的构造器。一个指定构造器将初始化类中提供的所有属性(就是需要确保所有非可选属性都有值)，并根据父类链往上调用父类的构造器来实现父类的初始化。

每一个类都必须拥有至少一个指定构造器。在某些情况下，许多类通过继承了父类中的指定构造器而满足了这个条件。

> 指定构造器只能调用父类的构造器，不能调用自己的其他构造器

##### 便利构造器

*便利构造器*（convenience initializers）是类中比较次要的、辅助型的构造器。你可以定义便利构造器来调用同一个类中的指定构造器，并为其参数提供默认值。你也可以定义便利构造器来创建一个特殊用途或特定输入值的实例。

你应当只在必要的时候为类提供便利构造器，比方说某种情况下通过使用便利构造器来快捷调用某个指定构造器，能够节省更多开发时间并让类的构造过程更清晰明了。

> 便利构造器只能调用自己的其他构造器，不能调用父类的构造器

#### 指定构造器和便利构造器的语法

类的**指定构造器**的写法跟值类型简单构造器一样：

```swift
init(parameters) {
    statements
}
```

**便利构造器**也采用相同样式的写法，但需要在`init`关键字之前放置`convenience`关键字，并使用空格将它们俩分开：

```swift
convenience init(parameters) {
    statements
}
```

#### 类的构造器代理规则

- 指定构造器必须总是向上代理
- 便利构造器必须总是横向代理

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/swift_init1.png?raw=true)



#### 两段式构造过程

Swift 中类的构造过程包含两个阶段。第一个阶段，每个存储型属性被引入它们的类指定一个初始值。当每个存储型属性的初始值被确定后，第二阶段开始，它给每个类一次机会，在新实例准备使用之前进一步定制它们的存储型属性。

**上面的话的大致意思是：先把子类特有的属性初始化完成后，再调用父类的构造函数初始化父类的属性。初始化父类的属性完后，再修改父类的属性。**

Swift 编译器将执行 4 种有效的安全检查，以确保两段式构造过程能不出错地完成：

##### 安全检查1

指定构造器必须保证它所在类引入的所有属性都必须先初始化完成，之后才能将其它构造任务向上代理给父类中的构造器。

如上所述，一个对象的内存只有在其所有存储型属性确定之后才能完全初始化。为了满足这一规则，指定构造器必须保证它所在类引入的属性在它往上代理之前先完成初始化。

> 就是在 super 之前，先要让子类的所有属性都有默认值

##### 安全检查2

指定构造器必须先向上代理调用父类构造器，然后再为继承的属性设置新值。如果没这么做，指定构造器赋予的新值将被父类中的构造器所覆盖。

> 就是第一步之后先super，然后修改其属性

##### 安全检查3

便利构造器必须先代理调用同一类中的其它构造器，然后再为任意属性赋新值。如果没这么做，便利构造器赋予的新值将被同一类中其它指定构造器所覆盖。

> 便利构造器先调用其他构造器，再修改其属性

##### 安全检查4

构造器在第一阶段(super 完成后)构造完成之前，不能调用任何实例方法，不能读取任何实例属性的值，不能引用`self`作为一个值。

> 只有 super 完成后，才能使用实例的属性方法			

类实例在第一阶段结束以前并不是完全有效的。只有第一阶段完成后，该实例才会成为有效实例，才能访问属性和调用方法。

##### 总结

到这里我们就可以将指定构造器和便利构造器的职责划分一下了。

**指定构造器实现的就是阶段一，先设置自身的非可选属性，然后调用 super**

**便利构造器实现的就是阶段二，在指定构造器构造完成后进行一些属性值的修改**

一般都是外部调用便利构造器，再由便利构造器调用指定构造器



#### 构造器的继承和重写

跟 Objective-C 中的子类不同，**Swift 中的子类默认情况下不会继承父类的构造器。**

关于指定构造器，当你在编写一个和父类中指定构造器相匹配的子类构造器时，你实际上是在重写父类的这个指定构造器。因此，你必须在定义子类构造器时带上`override`修饰符。即使你重写的是系统自动提供的默认构造器，也需要带上`override`修饰符。**你也可以将指定构造器重写为便利构造器。**

相反，关于便利构造器，如果你编写了一个和父类便利构造器相匹配的子类构造器，由于**子类不能直接调用父类的便利构造器**，因此，严格意义上来讲，**你的子类并未对一个父类构造器提供重写，而是直接覆盖**。最后的结果就是，你在**子类中“重写”一个父类便利构造器时，不需要加`override`前缀，即虽然类型相同，但不是同一个方法**。



#### 构造器的自动继承

子类在默认情况下不会继承父类的构造器。但是如果满足特定条件，父类构造器是可以被自动继承的。在实践中，这意味着对于许多常见场景你不必重写父类的构造器，并且可以在安全的情况下以最小的代价继承父类的构造器：

1. 如果子类没有定义任何指定构造器，它将自动继承所有父类的构造器（包括指定构造器和便利构造器）。
2. 如果子类提供了所有父类指定构造器的实现，它将自动继承所有父类的便利构造器。

即使你在子类中添加了很多的便利构造器，这两条规则仍然适用。对于规则 2，**子类可以将父类的指定构造器实现为便利构造**。



### 关于构造器继承与重写的总结

前面基本上以及都提及了什么时候能够能什么时候不能继承与重写，以及为什么。再强调一下之后的所有原则都是围绕：**如果哪里用到了未赋值的属性，就会产生异常**

> 一个指定构造器不能调用同一个类内的其他指定构造器

这个原因很简单。如果指定构造器能够调用同类的其他指定构造器，那还要便利构造器干嘛？

> 子类的便利构造器不能调用父类的构造器

因为便利构造器的目的就是为类中的指定构造器提供辅助的，即其作用于横向。如果其能够调用父类的构造器，那么便利构造器和指定构造器就没有区别了，也就是说没有必要提供便利构造器这个概念了。

> 子类的指定构造器不能调用父类的便利构造器

如果子类的指定构造器可以调用父类的便利构造器，那么考虑一种情况：子类的指定构造器重写了父类的指定构造器，并且父类的便利构造器会调用该指定构造器。这种情况下，由于重写，就会造成循环调用：

![调用循环](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/swift_init2.png?raw=true)

> 为什么会有两段式构造过程

和其他的语言不同：一般的语言不是第一句话都是调用 `super` 么，然后再自己修改值。Swift 则必须先初始化子类的属性后，才能进行后面的 `super`。为什么一定要这样？因为其他语言可以先将子类的属性默认设为0或者 `nil`，而 Swift 不行，不会为非可选属性设置初值。

安全检查2，3，4的目的就知道了，这三种情况都有可能用到未赋值的非可选属性（让某个属性等于某个未赋值的属性；在某个方法中使用了未赋值的属性）

安全检查1中，为什么一定要让赋初值在 super 前呢？因为父类的调用也有可能使用到未赋初值的子类属性。比如子类重写了某个父类的方法，然后父类的初始化方法中正好调用了这个方法。具体可见[详见](https://stackoverflow.com/questions/34307208/whats-the-purpose-of-initializing-property-values-before-calling-super-designat/34307391#34307391)

> 子类默认情况下不会继承父类的构造器

继承的构造器不会为子类的非可选属性设置初始值，如果哪里使用到了这个属性，就将产生异常。所以子类一般情况下不会继承父类的构造器。

> 子类中有与父类同名的便利构造器，不算重写，不需要加上 override

重写的意义是可以在子类中通过 super 调用父类的同名方法，如果不需要调用父类的该方法，那么不如直接覆盖。这里的便利构造器就是**覆盖，而不是重写**，因为不存在子类调用父类便利构造器的情况。

> 父类中的指定构造器，子类可以将其重写为指定构造器，也可以将其重写为便利构造器

和上面不同的是，父类的指定构造器是会被调用的，所以无论你在子类中将其实现为指定构造器还是便利构造器，都是重写，需要加上 override

> 为什么子类没有实现指定构造器，或者重写了所有指定构造器，就能继承父类所有的构造器？

前面说过，默认不能继承是因为可能存在子类调用父类继承过来的方法后，访问未赋初始值的属性，产生异常。这里能够继承是因为：

1. 没有实现任意指定构造器，说明子类没有任何没有初始值的属性
2. 实现了所有指定构造器，说明子类将所有没有初始值的属性都已经赋了初值



### 可失败构造器

如果一个类、结构体或枚举类型的对象，在构造过程中有可能失败，则为其定义一个可失败构造器。这里所指的“失败”是指，如给构造器传入无效的参数值，或缺少某种所需的外部资源，又或是不满足某种必要的条件等。

为了妥善处理这种构造过程中可能会失败的情况。你可以在一个类，结构体或是枚举类型的定义中，添加一个或多个可失败构造器。其语法为在`init`关键字后面添加问号(`init?`)。

> 可失败构造器的参数名和参数类型，不能与其它非可失败构造器的参数名，及其参数类型相同。

可失败构造器会创建一个类型为自身类型的可选类型的对象。你通过`return nil`语句来表明可失败构造器在何种情况下应该“失败”。

> 严格来说，构造器都不支持返回值。因为构造器本身的作用，只是为了确保对象能被正确构造。因此你只是用`return nil`表明可失败构造器构造失败，而不要用关键字`return`来表明构造成功。

下例中，定义了一个名为`Animal`的结构体，检查传入参数是否是空字符串。如果是空字符串，那么构造失败。否则，`species`属性被赋值，构造成功。

> 其实就是允许在某些自己设定的情况下，构造返回 nil

```swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
```

你可以通过该可失败构造器来构建一个`Animal`的实例，并检查构造过程是否成功：

```swift
let someCreature = Animal(species: "Giraffe")
// someCreature 的类型是 Animal? 而不是 Animal

if let giraffe = someCreature {
    print("An animal was initialized with a species of \(giraffe.species)")
}
// 打印 "An animal was initialized with a species of Giraffe"

let anonymousCreature = Animal(species: "")
// anonymousCreature 的类型是 Animal?, 而不是 Animal

if anonymousCreature == nil {
    print("The anonymous creature could not be initialized")
}
// 打印 "The anonymous creature could not be initialized"
```

#### 枚举类型的可失败构造器

可以通过一个带一个或多个参数的可失败构造器来获取枚举类型中特定的枚举成员。如果提供的参数无法匹配任何枚举成员，则构造失败。

```swift
enum TemperatureUnit {
    case Kelvin, Celsius, Fahrenheit
    init?(symbol: Character) {
        switch symbol {
        case "K":
            self = .Kelvin
        case "C":
            self = .Celsius
        case "F":
            self = .Fahrenheit
        default:
            return nil
        }
    }
}
```

#### 带原始值的枚举类型的可失败构造器

带原始值的枚举类型会**自带一个可失败构造器`init?(rawValue:)`**，该可失败构造器有一个名为`rawValue`的参数，其类型和枚举类型的原始值类型一致，如果该参数的值能够和某个枚举成员的原始值匹配，则该构造器会构造相应的枚举成员，否则构造失败。

因此上面的`TemperatureUnit`的例子可以重写为：

```swift
enum TemperatureUnit: Character {
    case Kelvin = "K", Celsius = "C", Fahrenheit = "F"
}

let fahrenheitUnit = TemperatureUnit(rawValue: "F")
if fahrenheitUnit != nil {
    print("This is a defined temperature unit, so initialization succeeded.")
}
// 打印 "This is a defined temperature unit, so initialization succeeded."

let unknownUnit = TemperatureUnit(rawValue: "X")
if unknownUnit == nil {
    print("This is not a defined temperature unit, so initialization failed.")
}
// 打印 "This is not a defined temperature unit, so initialization failed."
```

#### 构造失败的传递

类，结构体，枚举的可失败构造器可以横向代理到类型中的其他可失败构造器。类似的，子类的可失败构造器也能向上代理到父类的可失败构造器。

无论是向上代理还是横向代理，如果你代理到的其他可失败构造器触发构造失败，整个构造过程将立即终止，接下来的任何构造代码不会再被执行。

> 可失败构造器也可以代理到其它的非可失败构造器。通过这种方式，你可以**增加一个可能的失败状态到现有的构造过程中**。

#### 重写一个可失败构造器

父类的可失败构造器可被子类的可失败构造器重写，也可被子类的非可失败构造器重写。但是反过来，父类的非可失败构造器不能被子类的可失败构造器重写。

> 为什么会有这样的规定？试想一下你正在子类中用一个可失败构造器重写父类非可失败构造器，编译器是肯定会报错的，那么你应该怎么做？你需要将子类中可失败的情况移到父类中去。这就体现了苹果的设计意图了。诸如字符串为空等造成构造失败的情况是共性的，苹果希望你将这些情况放在父类中判断。至于子类中允许这样的发生的话，就再用非可失败构造器重写。

一个重写可失败构造器的例子：

```swift
class Document {
    var name: String?
    // 该构造器创建了一个 name 属性的值为 nil 的 document 实例
    init() {}
    // 该构造器创建了一个 name 属性的值为非空字符串的 document 实例
    init?(name: String) {
        self.name = name
        if name.isEmpty { return nil }
    }
}
```

可以**在子类的非可失败构造器中使用强制解包来调用父类的可失败构造器**。比如，下面的`UntitledDocument`子类的`name`属性的值总是`"[Untitled]"`，它在构造过程中使用了父类的可失败构造器`init?(name:)`：

```swift
class UntitledDocument: Document {
    override init() {
        super.init(name: "[Untitled]")!
    }
}
```

在这个例子中，如果在调用父类的可失败构造器`init?(name:)`时传入的是空字符串，那么强制解包操作会引发运行时错误。

#### 可失败构造器 init!

通常来说我们通过在`init`关键字后添加问号的方式（`init?`）来定义一个可失败构造器，但你也可以通过在`init`后面添加惊叹号的方式来定义一个可失败构造器（`init!`），该可失败构造器将会构建一个对应类型的隐式解包可选类型的对象。

你可以在`init?`中代理到`init!`，反之亦然。你也可以用`init?`重写`init!`，反之亦然。你还可以用`init`代理到`init!`，不过，**一旦`init!`构造失败，则会触发一个断言。**

> 这里 `self.init!()` 其实等价于 `self.init?()!`。这样写更方便一些。
>
> 这和变量的强制解包有一定的区别。**变量的强制解包是变量在其他地方以可选的形式存在，可以为nil，而某些地方需要表现为非可选，不能为nil**。而这里你是创建的时候就进行了隐式解包，其他地方根本不会用到其为 nil 的形式。如果你想创建一个非空的实例，为什么不直接用非可选构造器？
>
> 因为**这里就是想要在某些构造失败的情况下触发断言。**等同于非可选的 init 方法中，在某些情况下手动抛出异常。 

### 必要构造器

在类的构造器前添加`required`修饰符表明所有该类的子类都必须实现该构造器：

```swift
class SomeClass {
    required init() {
        // 构造器的实现代码
    }
}
```

在子类重写父类的必要构造器时，必须在子类的构造器前也添加`required`修饰符，表明该构造器要求也应用于继承链后面的子类。在重写父类中必要的指定构造器时，不需要添加`override`修饰符：

```swift
class SomeSubclass: SomeClass {
    required init() {
        // 构造器的实现代码
    }
}
```

> required 表示所有子类都必须实现这个构造器，而不是说，只要一个子类实现就可以了。

### 通过闭包或函数设置属性的默认值

如果某个存储型属性的默认值需要一些定制或设置，你可以使用闭包或全局函数为其提供定制的默认值。每当某个属性所在类型的新实例被创建时，对应的闭包或函数会被调用，而它们的返回值会当做默认值赋值给这个属性。

这种类型的闭包或函数通常会创建一个跟属性类型相同的临时变量，然后修改它的值以满足预期的初始状态，最后返回这个临时变量，作为属性的默认值。

```swift
class SomeClass {
    let someProperty: SomeType = {
        // 在这个闭包中给 someProperty 创建一个默认值
        // someValue 必须和 SomeType 类型相同
        return someValue
    }()
}
```

注意闭包结尾的大括号后面接了一对空的小括号。这用来告诉 Swift 立即执行此闭包。如果你忽略了这对括号，相当于将闭包本身作为值赋值给了属性，而不是将闭包的返回值赋值给属性。

> 如果你使用闭包来初始化属性，请记住在闭包执行时，实例的其它部分都还没有初始化。这意味着你不能在闭包里访问其它属性，即使这些属性有默认值。同样，你也不能使用隐式的`self`属性，或者调用任何实例方法。
>
> 如果一定想要用到self等怎么办，将该属性设置为 lazy。使用懒加载就能保证该属性一定在对象初始化完成后再初始化了

> 一定注意区分闭包和计算型属性的不同，闭包是个赋值操作，且最后有一个执行闭包的 `()`。计算型属性则是直接将计算方式写在属性后面

例如下面初始化一个西洋棋盘（黑白相间那种）：

```swift
struct Checkerboard {
    let boardColors: [Bool] = {
        var temporaryBoard = [Bool]()
        var isBlack = false
        for i in 1...8 {
            for j in 1...8 {
                temporaryBoard.append(isBlack)
                isBlack = !isBlack
            }
            isBlack = !isBlack
        }
        return temporaryBoard
    }()
    func squareIsBlackAtRow(row: Int, column: Int) -> Bool {
        return boardColors[(row * 8) + column]
    }
}

let board = Checkerboard()
print(board.squareIsBlackAtRow(0, column: 1))
// 打印 "true"
print(board.squareIsBlackAtRow(7, column: 7))
// 打印 "false"
```

