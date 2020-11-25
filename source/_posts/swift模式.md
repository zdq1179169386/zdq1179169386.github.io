---
title: swift 模式和模式匹配
date:  2019-06-10 18:17:51
tags: [Swift]

---
模式： 代表单个值或复合值的结构
swift 的模式分为两种： 一种能匹配任何类型的值，另一种在运行时匹配特定值时可能会失败
第一类模式可用于解构简单变量，常量和可选绑定中的值，此类包括通配符模式，标识符模式，以及包含前两种的值绑定模式和元组模式，你可以为这类模式指定一个类型，从而限制它们只能匹配某种特定类型的值。
第二类模式用于全模式匹配，这种情况下你试图匹配的值可能在运行时不存在。此类模式包括枚举用例模式，可选模式，表达式模式和类型转换模式。

<!--more-->

### 1, 通配符模式 
通配符模式会忽略需要匹配的值，_ 表示你不再使用这个值

``` bash
for _ in 1..<10 {
    print("-----")
}
```

### 2，标识符模式

``` bash
let someValue = 42
标识符模式匹配任何值，并将匹配的值和一个变量或常量绑定起来。例如，在上面的常量申明中，someValue是一个标识符模式，匹配了类型是Int的42。
当匹配成功时，42被绑定(赋值)给常量someValue。
当一个变量或常量申明的左边是标识符模式时，此时，标识符模式是隐式的值绑定模式(value-binding pattern)。
```
### 3，值绑定模式
值绑定模式绑定匹配的值到一个变量或常量。当绑定匹配值给常量时，用关键字let,绑定给变量时，用关键之var。

标识符模式包含在值绑定模式中，绑定新的变量或常量到匹配的值。例如，你可以分解一个元组的元素，并把每个元素绑定到相应的标识符模式中。

``` bash
let point = (2,3)
switch point {
 case let (x,y):
    print("point is \(x),\(y)")
    
}
```
### 4，元组模式
元组模式是逗号分隔的列表，包含一个或多个模式，并包含在一对圆括号中。元组模式匹配相应元组类型的值。

``` bash
let points = [(0,0),(1,0),(1,1),(1,2),(2,0)]
for (x,y) in points where y == 0 {
    print("\(x),\(y)")
}
```
### 5，枚举用例模式
枚举用例模式感觉和元组模式，有点像。

``` bash
enum Entities{
    case Soldier(x: Int,y: Int)
    case Tank(x:Int, y: Int)
    case Play(x: Int,y: Int)
}
let entities = [Entities.Soldier(x: 0, y: 0),Entities.Tank(x: 1, y: 1),Entities.Play(x: 2, y: 2)]
for e in entities {
    switch e {
    case let .Soldier(x,y):
        print("Soldier x is \(x),y is \(y)")
    case let .Tank(x,y):
        print("Tank x is \(x),y is \(y)")
    case let .Play(x,y):
        print("Play x is \(x),y is \(y)")
    default:
        print("---")
    }
}
```
### 6, 可选项模式
someValue 是字符串，也有可能是nil，通过模式匹配也可以帮它解包

```
let someValue : Int? = 42
if case let x? = someValue {
    print("\(x)")
}
let arrOptinals : [Int?] = [1,nil,3,4,nil]
for case let x? in arrOptinals {
    print("\(x)")
}
```
### 7，类型转换模式
is 模式，不会转换类型, as 模式 会转换类型

```
protocol Animal{
    var name : String {get}
}
struct Dog : Animal{
    var name: String {
        return "dog"
    }
    var speed : Int
}
struct Bird : Animal{
    var name: String {
        return "bird"
    }
    var height: Int
    
}
struct Fish : Animal{
    var name: String{
        return "fish"
    }
    var depth: Int
}
let animals:[Animal] = [Dog(speed: 20),Bird(height: 1000),Fish(depth: 500)]
for animal in animals {
    switch animal {
    case let dog as Dog:
        print("dog name \(dog.name),can run \(dog.speed)")
    case let bird as Bird:
        print("bird name \(bird.name),can fiy \(bird.height)")
    case is Fish :
        print("this is fish")
    default:
        print("unknown animal")
    }
}
```
### 8, 表达式模式
表达式模式代表了一个表达式的值。这个模式只出现在switch语句中的case标签中。
由表达式模式所代表的表达式用Swift标准库中的~=操作符与输入表达式的值进行比较。如果~=操作符返回true，则匹配成功。默认情况下，~=操作符使用==操作符来比较两个相同类型的值。它也可以匹配一个整数值与一个Range对象中的整数范围。

```
struct Teacher{
    var name: String
    var salary: Int
}
func ~= (pattern: Range<Int>,value: Teacher) -> Bool{
    return pattern.contains(value.salary)
}
let teacher = Teacher(name: "zdq", salary: 10000)
switch teacher {
case 0..<2000:
    print("拮据")
case 2000..<10000:
    print("滋润")
case 10000 ..< 20000:
    print("土豪")
default:
    print("")
}
```
### 9，常用的模式匹配
if-case

guard-case

for-case

[模式匹配第四弹：if case，guard case，for case](https://swift.gg/2016/06/06/pattern-matching-4/)

### 10，ES6的解构 和swift 的模式匹配
之前看 ES6 里面的解构赋值很方便，感觉跟swift 的元组解构有点像, 比如:

js 的基础类型解构

```bash
let [a, b, c] = [1, 2, 3]
console.log(a, b, c) // 1, 2, 3
```
swift 的元组解构

```
let (x,y) = (1,2)
print(x,y) // 1, 2 
```

解构对象重命名

```
let { f1: rename, f2 } = { f1: 'test1', f2: 'test2' }
console.log(rename, f2) // test1, test2
```

swift 的元组解构

```
let (x,y) = (name: "zhangsan",age: 20)
print(x,y) // zhangsan 20
```
当然swift 当中只有元组可以解构，数组和字典都不行,比如下面的写法都是报错的

```
let [x,y,z] = [1,2,3] // error: expected pattern
```
```
let [name: value, f2] = ["name":"zhangsan","age":20] // error: expected pattern
```


参考 [详解 Swift 模式匹配](https://swift.gg/2015/10/27/swift-pattern-matching-in-detail/)
