title: Swift 3.1 语法学习（二）
date: 2017/2/22 14:07:13  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->



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

