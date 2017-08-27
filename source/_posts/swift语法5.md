title: Swift 3.1 语法学习（五）
date: 2017/2/22 14:07:18  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

## 析构过程

*析构器*只适用于类类型，当一个类的实例被释放之前，析构器会被立即调用。析构器用关键字`deinit`来标示，类似于构造器要用`init`来标示。

### 析构过程原理

Swift 通过自动引用计数（ARC）处理实例的内存管理。通常当你的实例被释放时不需要手动地去清理。但是，当使用自己的资源时，你可能需要进行一些额外的清理。例如，如果创建了一个自定义的类来打开一个文件，并写入一些数据，你可能需要在类实例被释放之前手动去关闭该文件。

在类的定义中，每个类最多只能有一个析构器，而且析构器不带任何参数，如下所示：

```swift
deinit {
    // 执行析构过程
}
```

析构器是在实例释放发生前被自动调用。你不能主动调用析构器。子类继承了父类的析构器，并且在子类析构器实现的最后，父类的析构器会被自动调用。即使子类没有提供自己的析构器，父类的析构器也同样会被调用。

因为直到实例的析构器被调用后，实例才会被释放，所以**析构器可以访问实例的所有属性**，并且可以根据那些属性可以修改它的行为（比如查找一个需要被关闭的文件）。

> 其实就是一个在对象销毁前的回调，你可以在对象销毁前做任何你想做的事。

## 自动引用计数

> 引用计数仅仅应用于类的实例。结构体和枚举类型是值类型，不是引用类型，也不是通过引用的方式存储和传递。

### 解决实例间的循环强引用

Swift 提供了两种办法用来解决你在使用类的属性时所遇到的循环强引用问题：弱引用（weak reference）和无主引用（unowned reference）。

对于生命周期中会变为`nil`的实例使用弱引用。相反地，对于初始化赋值后再也不会被赋值为`nil`的实例，使用无主引用。

#### 弱引用

在实例的生命周期中，如果某些时候引用没有值，那么弱引用可以避免循环强引用。

> 弱引用必须被声明为变量，表明其值能在运行时被修改。**弱引用不能被声明为常量**。
>
> 弱引用只能修饰类对象，不能修饰引用传递对象，即类对象，不能修饰值传递引用，即不能修饰基本类型，结构体，数组等。(涉及到引用了，肯定是要修饰引用传递对象啊)

因为弱引用可以没有值，你必须将每一个弱引用声明为可选类型。因为弱引用不会保持所引用的实例，即使引用存在，实例也有可能被销毁。因此，ARC 会在引用的实例被销毁后自动将其赋值为`nil`。你可以像其他可选值一样，检查弱引用的值是否存在，你将永远不会访问已销毁的实例的引用。

弱引用的声明在变量前加上 `weak` 关键字：

```swift
class Apartment {
    let unit: String
    init(unit: String) { self.unit = unit }
    weak var tenant: Person?
    deinit { print("Apartment \(unit) is being deinitialized") }
}
```

#### 无主引用

和弱引用类似，无主引用不会牢牢保持住引用的实例。和弱引用不同的是，**无主引用是永远有值的**。因此，无主引用总是被定义为非可选类型（non-optional type）。你可以在声明属性或者变量时，在前面加上关键字`unowned`表示这是一个无主引用。

由于无主引用是非可选类型，你不需要在使用它的时候将它展开。无主引用总是可以被直接访问。不过 ARC 无法在实例被销毁后将无主引用设为`nil`，因为非可选类型的变量不允许被赋值为`nil`。

> 如果你试图在实例被销毁后，访问该实例的无主引用，会触发运行时错误。使用无主引用，你必须确保引用始终指向一个未销毁的实例。(A 中有一个属性 B为无主引用，那么就要确保 B 没有被回收)
>
> 无主应用也只能修饰引用传递对象

#### 无主引用以及隐式解析可选属性

上面弱引用和无主引用的例子涵盖了两种常用的需要打破循环强引用的场景。

两个属性的值都允许为`nil`，并会潜在的产生循环强引用。这种场景最适合用弱引用来解决。一个属性的值允许为`nil`，而另一个属性的值不允许为`nil`，这也可能会产生循环强引用。这种场景最适合通过无主引用来解决。

然而，存在着第三种场景，在这种场景中，两个属性都必须有值，并且初始化完成后永远不会为`nil`。在这种场景中，需要一个类使用无主属性，而另外一个类使用隐式解析可选属性。

> 非可选 A 和 非可选 B 相互引用，但是由于“两段式构造过程”，比如要初始化 A，那么必须要先初始化B，但是B要初始化完成必须先初始化A，这就像死锁一样。所以为了打破这种循环，必须用隐式可选属性，既能表示不能为空，又能不影响初始化的进行。

```swift
class Country {
    let name: String
    var capitalCity: City!
    init(name: String, capitalName: String) {
        self.name = name
        self.capitalCity = City(name: capitalName, country: self)
    }
}

class City {
    let name: String
    unowned let country: Country
    init(name: String, country: Country) {
        self.name = name
        self.country = country
    }
}

var country = Country(name: "Canada", capitalName: "Ottawa")
print("\(country.name)'s capital city is called \(country.capitalCity.name)")
// 打印 “Canada's capital city is called Ottawa”
```

上面如果使用无主引用，无主引用是常量类型，并且对象必须要在其内部常量属性都有了初值之后才能使用 self 属性，所以 `country:self` 就会产生异常。必须要使用变量类型来替代原来的无主引用。

> 无主引用和隐式解析的区别在于，无主引用永远不能为 nil，而隐式解析值要保证用到的时候不为 nil



### 闭包引起的循环强引用

这个闭包体中可能访问了实例的某个属性，例如`self.someProperty`，或者闭包中调用了实例的某个方法，例如`self.someMethod()`。这两种情况都导致了闭包“捕获”`self`，从而产生了循环强引用:

```swift
class HTMLElement {

    let name: String
    let text: String?

    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }

    deinit {
        print("\(name) is being deinitialized")
    }

}
```

> `asHTML`声明为`lazy`属性，因为只有当元素确实需要被处理为 HTML 输出的字符串时，才需要使用`asHTML`。也就是说，在默认的闭包中可以使用`self`，因为只有当初始化完成以及`self`确实存在后，才能访问`lazy`属性。如果没有 `lazy` 修饰，那么是不能编译通过的。

#### 解决闭包引起的循环强引用

获列表中的每一项都由一对元素组成，一个元素是`weak`或`unowned`关键字，另一个元素是类实例的引用（例如`self`）或初始化过的变量（如`delegate = self.delegate!`）。这些项在方括号中用逗号分开。

如果闭包有参数列表和返回类型，把捕获列表放在它们前面：

```swift
lazy var someClosure: (Int, String) -> String = {
    [unowned self, weak delegate = self.delegate!] (index, stringToProcess) in
    // 这里是闭包的函数体
}
```

上面闭包的参数类型由于可以通过类型推断得到，因此都省略了。如果闭包没有指明参数列表，那么可以把捕获列表和关键字`in`放在闭包最开始的地方：

```swift
lazy var someClosure: () -> String = {
    [unowned self, weak delegate = self.delegate!] in
    // closure body goes here
}
```



## 可选链式调用

可选链式调用（Optional Chaining）是一种可以在当前值可能为`nil`的可选值上请求和调用属性、方法及下标的方法。如果可选值有值，那么调用就会成功；如果可选值是`nil`，那么调用将返回`nil`。多个调用可以连接在一起形成一个调用链，如果其中任何一个节点为`nil`，整个调用链都会失败，即返回`nil`。

### 使用可选链式调用代替强制展开

通过在想调用的属性、方法、或下标的可选值（optional value）后面放一个问号（`?`），可以定义一个可选链。这一点很像在可选值后面放一个叹号（`!`）来强制展开它的值。它们的主要区别在于当可选值为空时可选链式调用只会调用失败，然而强制展开将会触发运行时错误。

可选链式调用的返回结果与原本的返回结果具有相同的类型，但是被包装成了一个可选值。例如，使用可选链式调用访问属性，当可选链式调用成功时，如果属性原本的返回结果是`Int`类型，则会变为`Int?`类型。

下面的例子很清楚活命可选链式调用：

```swift
class Person {
    var residence: Residence?
}

class Residence {
    var numberOfRooms = 1
}

let john = Person()

let roomCount = john.residence!.numberOfRooms
// 这会引发运行时错误

if let roomCount = john.residence?.numberOfRooms {
    print("John's residence has \(roomCount) room(s).")
} else {
    print("Unable to retrieve the number of rooms.")
}
// Prints "John's residence has 1 room(s)."
```

不只是属性，方法和下标也是可选链式调用的。



### 连接多层可选链式调用

有时候会嵌套使用，如：

```swift
if let johnsStreet = john.residence?.address?.street {
    print("John's street name is \(johnsStreet).")
} else {
    print("Unable to retrieve the address.")
}
// 打印 “Unable to retrieve the address.”
```

返回的仍然是 `String?` 而不是 `String??`

### 在方法的可选返回值上进行可选链式调用

如果要在方法的返回值上进行可选链式调用，在方法的圆括号后面加上问号即可：

```swift
if let beginsWithThe =
    john.residence?.address?.buildingIdentifier()?.hasPrefix("The") {
        if beginsWithThe {
            print("John's building identifier begins with \"The\".")
        } else {
            print("John's building identifier does not begin with \"The\".")
        }
}
// 打印 “John's building identifier begins with "The".”
```

## 错误处理

### 表示并抛出错误

在 Swift 中，错误用符合`Error`协议的类型的值来表示。这个空协议表明该类型可以用于错误处理。

```swift
enum VendingMachineError: Error {
    case invalidSelection
    case insufficientFunds(coinsNeeded: Int)
    case outOfStock
}

throw VendingMachineError.insufficientFunds(coinsNeeded: 5)
```

> 这里枚举中的 `coninsNeeded` 就是前面讲枚举的时候的关联值，只是设置了参数名，比如下面枚举的元组情况可能会用到



## 类型转换

### 类型检查

用类型检查操作符（`is`）来检查一个实例是否属于特定子类型。若实例属于那个子类型，类型检查操作符返回 `true`，否则返回 `false`。

```swift
for item in library {
    if item is Movie {
        movieCount += 1
    } else if item is Song {
        songCount += 1
    }
}
```

这里只截取了代码的部分。演示了 `is` 的用法。

### 类型转换

用类型转化你操作符 `as` 来转换类型。 `as`  应用条件有2种情况：

1.  和 `as` 右边类型一致
2. 是右边类型的子类(向上转型)

### 向下转型

某类型的一个常量或变量可能在幕后实际上属于一个子类。当确定是这种情况时，你可以尝试向下转到它的子类型，用类型转换操作符（`as?` 或 `as!`）。

因为向下转型可能会失败，类型转型操作符带有两种不同形式。条件形式（conditional form）`as?` 返回一个你试图向下转成的类型的可选值（optional value）。强制形式 `as!` 把试图向下转型和强制解包（force-unwraps）转换结果结合为一个操作。

在这个示例中，数组中的每一个 `item` 可能是 `Movie` 或 `Song`。事前你不知道每个 `item` 的真实类型，所以这里使用条件形式的类型转换（`as?`）去检查循环里的每次下转：

```swift
for item in library {
    if let movie = item as? Movie {
        print("Movie: '\(movie.name)', dir. \(movie.director)")
    } else if let song = item as? Song {
        print("Song: '\(song.name)', by \(song.artist)")
    }
}

// Movie: 'Casablanca', dir. Michael Curtiz
// Song: 'Blue Suede Shoes', by Elvis Presley
// Movie: 'Citizen Kane', dir. Orson Welles
// Song: 'The One And Only', by Chesney Hawkes
// Song: 'Never Gonna Give You Up', by Rick Astley
```

###  Any 和 AnyObject 的类型转换

Swift 为不确定类型提供了两种特殊的类型别名：

- `AnyObject` 可以表示任何类类型的实例。
- `Any` 可以表示任何类型，包括函数类型。

例如下面使用 `Any` 的例子：

```swift
var things = [Any]()
 
things.append(0)
things.append(0.0)
things.append(42)
things.append(3.14159)
things.append("hello")
things.append((3.0, 5.0))
things.append(Movie(name: "Ghostbusters", director: "Ivan Reitman"))
things.append({ (name: String) -> String in "Hello, \(name)" })
```

现在结合 `is` 和 `as` 使用：

```swift
for thing in things {
    switch thing {
    case 0 as Int:
        print("zero as an Int")
    case 0 as Double:
        print("zero as a Double")
    case let someInt as Int:
        print("an integer value of \(someInt)")
    case let someDouble as Double where someDouble > 0:
        print("a positive double value of \(someDouble)")
    case is Double:
        print("some other double value that I don't want to print")
    case let someString as String:
        print("a string value of \"\(someString)\"")
    case let (x, y) as (Double, Double):
        print("an (x, y) point at \(x), \(y)")
    case let movie as Movie:
        print("a movie called \(movie.name), dir. \(movie.director)")
    case let stringConverter as (String) -> String:
        print(stringConverter("Michael"))
    default:
        print("something else")
    }
}
 
// zero as an Int
// zero as a Double
// an integer value of 42
// a positive double value of 3.14159
// a string value of "hello"
// an (x, y) point at 3.0, 5.0
// a movie called Ghostbusters, dir. Ivan Reitman
// Hello, Michael
```

> case 还能用 `is 类型` 判断是某一类类型；`let xx as 类型` 判断是某一种类型。
>
> 厉害了

## 嵌套类型

枚举常被用于**为特定类或结构体**实现某些功能。类似的，枚举可以方便的定义工具类或结构体，从而为某个复杂的类型所使用。为了实现这种功能，Swift 允许你定义嵌套类型，可以在支持的类型中定义嵌套的枚举、类和结构体。

要在一个类型中嵌套另一个类型，将嵌套类型的定义写在其外部类型的`{}`内，而且可以根据需要定义多级嵌套。



### 嵌套类型实践

```swift
struct BlackjackCard {
    // 嵌套的 Suit 枚举
    enum Suit: Character {
        case Spades = "♠", Hearts = "♡", Diamonds = "♢", Clubs = "♣"
    }
    
    // 嵌套的 Rank 枚举
    enum Rank: Int {
        case Two = 2, Three, Four, Five, Six, Seven, Eight, Nine, Ten
        case Jack, Queen, King, Ace
        struct Values {
            let first: Int, second: Int?
        }
        var values: Values {
            switch self {
            case .Ace:
                return Values(first: 1, second: 11)
            case .Jack, .Queen, .King:
                return Values(first: 10, second: nil)
            default:
                return Values(first: self.rawValue, second: nil)
            }
        }
    }
    
    // BlackjackCard 的属性和方法
    let rank: Rank, suit: Suit
    var description: String {
        var output = "suit is \(suit.rawValue),"
        output += " value is \(rank.values.first)"
        if let second = rank.values.second {
            output += " or \(second)"
        }
        return output
    }
}

let theAceOfSpades = BlackjackCard(rank: .Ace, suit: .Spades)
print("theAceOfSpades: \(theAceOfSpades.description)")
// 打印 “theAceOfSpades: suit is ♠, value is 1 or 11”
```

注意看这个例子的应用部分。尽管`Rank`和`Suit`嵌套在`BlackjackCard`中，但它们的**类型仍可从上下文中推断出来**，所以在初始化实例时能够单独通过成员名称（`.Ace`和`.Spades`）引用枚举实例。

> 如果没有类型推断，需要将代码补全的话，需要写成：`BlackjackCard.Rank.Ace` 的形式。

### 引用嵌套类型

在外部引用嵌套类型时，在嵌套类型的类型名前加上其外部类型的类型名作为前缀。就和上面说的一样：

```swift
let heartsSymbol = BlackjackCard.Suit.Hearts.rawValue
// 红心符号为 “♡”
```



