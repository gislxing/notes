# Kotlin 学习笔记

## 声明变量

`Kotlin` 提供了两个关键字声明变量：

- `val` 声明一个**不变的变量**（**命名值**或**常量**），在初始化之后不能更改
- `var` 声明一个**可变的变量**，可以更改

让我们声明一个不可变的变量`language`，并使用 string 对其进行初始化`"Kotlin"`。

```kotlin
// 此变量在初始化后无法修改，因为它已声明为val
val language = "Kotlin"
```

可变变量（用关键字`var`声明）有一个限制。您只能重新分配与初始值相同类型的值。因此，下面的代码段是不正确的：

```kotlin
var number = 10
number = 11 // ok
number = "twelve" // an error here!
```

**尽可能使用 `val` 声明不可变变量，如果 `val` 不合适时再使用 `var` 声明变量**

## 数据类型

**创建变量后，它就具有类型，并且以后不能更改其类型**

变量都具有一种**类型**，该**类型**决定了可能的操作，可以对该变量执行的**类型**以及可以存储在该变量中的值

### 变量类型

在声明变量时设置其类型：

```kotlin
val text = "Hello, I am studying Kotlin now."
val n = 1
```

在这种情况下，Kotlin知道这`text`是一个字符串，并且`n`是一个数字。Kotlin会自动确定两个变量的类型。这种机制称为**类型推断**

### 用类型推断声明变量

```kotlin
val/var identifier = initialization
```

在声明变量时指定其类型：

```kotlin
val/var identifier: Type = initialization 
```

请注意，类型的名称始终以大写字母开头

```kotlin
val text: String = "Hello, I am studying Kotlin now."
val n: Int = 1
```

示例

```kotlin
// 错误，因为此时类型推断无法确定变量的类型
val greeting // error
greeting = "hello"

// 正确，指定变量类型
val greeting: String // ok
greeting = "hello"
```

### 基本数据类型

#### 整型

**整数**（0，1，2，...）是由以下四种类型表示：`Long`，`Int`，`Short`，`Byte`（从最大到最小）

如果整数值包含很多数字，我们可以添加下划线(`_`)以将数字划分为块，以使该数字更具可读性：例如，`1_000_000`比起 `1000000` 更易于阅读

整数类型的范围计算为：-(2<sup>n-1</sup>) 至 (2<sup>n-1</sup>)-1，其中*n*是位数

- `Byte`：大小8位（1字节），范围-128至127
- `Short`：大小为16位（2个字节），范围为-32768至32767
- `Int`：大小为32位（4字节），范围为−(2<sup>31</sup>）至（2<sup>31</sup>) - 1
- `Long`：大小为64位（8字节），范围为−（2<sup>63</sup>）至（2<sup>63</sup>）−1

```kotlin
val zero = 0 // Int
val one = 1  // Int
val oneMillion = 1_000_000  // Int

val twoMillion = 2_000_000L // Long, because it is tagged with L
val bigNumber = 1_000_000_000_000_000 // Long, Kotlin automatically choose it (Int is too small)
val ten: Long = 10 // Long, because the type is specified

val shortNumber: Short = 15 // Short, because the type is specified
val byteNumber: Byte = 15   // Byte, because the type is specified
```

#### 浮点类型

表示带有小数部分的数字。`Kotlin` 有两种类型：`Double`（64位）和`Float`（32位）。`Floag` 最大能存储 `6-7` 位小数，`Double` 最大能存储 `14-16` 位小数。通常，实际开发中使用`Double` 类型

```kotlin
val pi = 3.1415  // Double
val e = 2.71828f // Float, because it is tagged with f
val fraction: Float = 1.51 // Float, because the type is specified
```

要显示数字类型（包括`Double`和`Float`）的最大值和最小值，您需要输入类型名称，后跟一个点，然后输入`MIN_VALUE`或`MAX_VALUE`

```kotlin
println(Int.MIN_VALUE)  // -2147483648
println(Int.MAX_VALUE)  // 2147483647
println(Long.MIN_VALUE) // -9223372036854775808
println(Long.MAX_VALUE) // 9223372036854775807
```

也可以获取整数类型的大小，以字节或位为单位（1字节= 8位）：

```kotlin
println(Int.SIZE_BYTES) // 4
println(Int.SIZE_BITS)  // 32
```

#### 字符类型

`Char`用于表示字母（大写和小写）、数字和其他符号。每个字符是用单引号引起来的单个字母

```kotlin
val lowerCaseLetter = 'a'
val upperCaseLetter = 'Q'
val number = '1'
val space = ' '
val dollar = '$'
```

#### 布尔类型

`Boolean` 类型只能存储两个值：`true`和`false`

```kotlin
val enabled = true
val bugFound = false
```

#### 字符串类型

`String`类型是用双引号引起来的一系列字符

```kotlin
val creditCardNumber = "1234 5678 9012 3456"
val message = "Learn Kotlin instead of Java."
```

### 类型转换

`Kotlin` 不进行自动类型转换，所有类型之间的转换必须显示的进行

#### 数字类型之间的转换

在实践中，常用的有三种数字类型：`Int`，`Long`，和`Double` 

将一种数字类型转换为另外一种数字类型需要进行类型转换：`toInt()`, `toLong()`, `toDouble()`等

```kotlin
val num: Int = 100

val res: Double = Math.sqrt(num.toDouble())
println(res) // 10.0

println(num) // 100, it is not modified
```

尽管`Char`不是数字类型，您仍可以根据字符代码（可以在Unicode表中找到）将数字转换为字符，反之亦然。该代码可以视为整数。

```kotlin
val n1: Int = 125
val ch: Char = n1.toChar() // '}'
val n2: Int = ch.toInt()   // 125
```

将较大的数字类型（例如`Long`或`Double`）转换为较小的数字类型（例如`Int`）也是可以的

```kotlin
val d: Double = 10.2
val n: Long = 15

val res1: Int = d.toInt() // 10
val res2: Int = n.toInt() // 15
```

但是，此转换可能会截断该值，因为`Long`和`Double`可以存储比 `Int` 更大的数字

```kotlin
val bigNum: Long = 100_000_000_000_000

// 类型溢出
val n: Int = bigNum.toInt() // 276447232; oops
```

#### 转换成字符串

`Kotlin` 提供了 `toString()`将某些内容转换为字符串

```kotlin
val n = 8     // Int
val d = 10.09 // Double
val c = '@'   // Char
val b = true  // Boolean

val s1 = n.toString() // "8"
val s2 = d.toString() // "10.09"
val s3 = c.toString() // "@"
val s4 = b.toString() // "true"
```

字符串可以转换为数字或布尔值（但不能转换为单个字符）

```kotlin
val n = "8".toInt() // Int
val d = "10.09".toDouble() // Double
val b = "true".toBoolean() // Boolean
```

如果数字的字符串表示格式无效，则在转换过程中将发生异常

如果将字符串转换为布尔值，则不会发生任何错误。如果字符串为 `"true"`（忽略大小写），则转换结果为`true`，否则为`false`

```kotlin
val b1 = "false".toBoolean() // false
val b2 = "tru".toBoolean()   // false
val b3 = "true".toBoolean()  // true
val b4 = "TRUE".toBoolean()  // true
val b5 = "xxxx".toBoolean()  // false
```

## 函数

**在 `Kotlin` 中，所有函数都有返回值**

```kotlin
val result = println("text")
println(result) // kotlin.Unit
```

这里返回一个特殊的对象 `Unit`, 意思是：没有结果

