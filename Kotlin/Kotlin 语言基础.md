# Kotlin 语言基础

## [代码规范](https://kotlinlang.org/docs/reference/coding-conventions.html)

### 基本代码约定

1. 使用**4个空格**的缩进代码
2. 尽可能**省略分号`;`**
3. 将左大括号放在代码块开始的行结尾（opening curly brace）
4. 将右大括号放在下一行的开头

```kotlin
fun main() { // opening curly brace
    println("Hi") // this statement is offset by 4 spaces and has no `;` at the end
} // closing brace
```

### 变量命名

#### 变量命名规则

- 名称区分大小写
- 名称由字母，数字和下划线组成
- 名称不能以数字开头
- 一个名称不能是关键字（例如`val`，`var`，`fun`是非法的名称）

#### 变量命名规范

- 如果变量名是一个单词，则应使用小写字母
- 如果变量名包含多个单词，则应使用`lowerCamelCase`，即第一个单词应使用小写字母，其他所有单词都应使用大写字母开头
- 不能以`_` 开头，虽然这样是正确的
- 名称应该是有意义的

这些约定是可选的，但**强烈建议**您遵循它们

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

## 注释

注释是一个特殊的文本，编译器将忽略它

`Kotlin` 有三种注释

### 行尾注释

编译器将忽略从 `//` 开始到行结尾的所有文本

```kotlin
fun main() {
    // The line below will be ignored
    // println("Hello, World")

    // This prints the string "Hello, Kotlin"
    println("Hello, Kotlin")  // Here can be any comment
}
```

### 多行注释

编译器将忽略 `/*` 和 `*/` 之间的任何文本

```kotlin
fun main() {
    /* This is a single-line comment */
    /*  This is an example of
        a multi-line comment */

    /* All lines below will be ignored
        println("Hello")
        println("World")
    */
}
```

### 文档注释

编译器将忽略 `/**` 和 `*/` 之间的任何文本

`文档注释` 可以使用工具来自动生成源码的文档

通常，这些注释放在某些程序元素的声明之上

```kotlin
/**
 * The `main` function accepts string arguments from outside.
 *
 * @param args arguments from the command line.
 */
fun main(args: Array<String>) {
    // do nothing
}
```

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

如果整数值包含很多数字，我们可以添加下划线(`_`)以将数字划分为块，以使该数字更具可读性：例如，`1_000_000`比起 `1000000` 更易于阅读，下划线不能放在数字的开头或末尾，下划线也可以连续放多个

```kotlin
val bigNum = 100____00__000__00
```

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

注意：带浮点数的运算可能会产生不准确的结果

```kotlin
println(3.3 / 3) // prints 1.0999999999999999
```

#### 字符类型

`Char`用于表示字母（大写和小写）、数字和其他符号。每个字符是用单引号引起来的

```kotlin
val lowerCaseLetter = 'a'
val upperCaseLetter = 'Q'
val number = '1'
val space = ' '
val dollar = '$'
```

也可以通过使用[Unicode表](https://unicode-table.com/en/)中的十六进制码来创建 `Char` 类型。以`\u`开头

```kotlin
val ch = '\u0040' // it represents '@'
println(ch) // @
```

尽管我们使用字符序列来表示此类代码，但是代码本身恰好表示一个字符。

##### 检索字符

使用 `+` 和 `-` 运算符可以对 `Char` 类型进行整型运算以便获得 `Unicode` 顺序表中的下一个和上一个字符

```kotlin
val ch1 = 'b'
val ch2 = ch1 + 1 // 'c'
val ch3 = ch2 - 2 // 'a'
```

`++` 、 `--` 、 `+=` 、 `-=` 运算符都可以对 `Char` 类型进行运算

```kotlin
var ch = 'A'

ch += 10
println(ch)   // 'K'
println(++ch) // 'L'
println(++ch) // 'M'
println(--ch) // 'L'
```

不能对 `Char` 类型进行乘除运算

##### 转义字符

以反斜线 `\` 开始的特殊字符表示转义字符或者控制字符

- `'\n'` 换行符
- `'\t'` 制表符 `tab`
- `'\r'` 回车符
- `'\\'` 反斜线
- `'\''` 单引号
- `'\"'` 双引号

Here are several examples:

```java
print('\t') // makes a tab
print('a')  // prints 'a'
print('\n') // goes to a new line
print('c')  // prints 'c'
```

##### 关系运算

根据字符在 `Unicode` 表中的位置可以进行关系运算

```kotlin
println('a' < 'c')  // true
println('x' >= 'z') // false

println('D' == 'D') // true
println('Q' != 'q') // true, because capital and small letters are not the same

println('A' < 'a')  // true, because capital Latin letters are placed before small ones
```

##### 字符处理

字符类型包含一些常用的方法可直接使用

- `isDigit()` 如果字符是数字则返回 `true`否则返回 `false` 
- `isLetter()` 如果字符是字母则返回 `true` 否则返回 `false`
- `isLetterOrDigit()` 如果字符是数字或者字母则返回 `true` 否则返回 `false`;
- `isWhitespace()` 如果字符是空白字符(`' '` or `'\t'` or `'\n'`)则返回 `true` 否则返回 `false`
- `isUpperCase()` 如果字符是大写字母则返回 `true` 否则返回 `false`;
- `isLowerCase()` 如果字符是小写字母则返回 `true` 否则返回 `false`;
- `toUpperCase()` 返回字符的大写形式
- `toLowerCase()` 返回字符的小写形式

```kotlin
val one = '1'

val isDigit = one.isDigit()   // true
val isLetter = one.isLetter() // false

val capital = 'A'
val small = 'e'

val isLetterOrDigit = capital.isLetterOrDigit() // true

val isUpperCase = capital.isUpperCase() // true
val isLowerCase = capital.isLowerCase() // false

val capitalE = small.toUpperCase() // 'E'
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

##### 访问字符

字符串的元素是可以通过其索引访问的单个字符。字符串的第一个元素的索引为 `0` 

```kotlin
val greeting = "Hello"

val first = greeting[0]  // 'H'
val second = greeting[1] // 'e'
val five = greeting[4]   // 'o'
```

最后一个元素的索引等于字符串的长度减去`1` 

```kotlin
val last = greeting[greeting.length - 1] // 'o'
val prelast = greeting[greeting.length - 2] // 'l'
```

`Kotlin` 提供了几种方便的方式来访问字符串的第一个和最后一个字符

```kotlin
println(greeting.first())   // 'H'
println(greeting.last())    // 'o'
println(greeting.lastIndex) // 4
```

##### 不可变性

字符串是**不可变的**， 这意味着一旦创建，字符串便始终相同。不能修改字符串的元素

```kotlin
val str = "string"
str[3] = 'o' // an error here!
```

但这一项有效：

```kotlin
var str = "string"
// 给字符串重新复制
str = "strong"
```

如果需要修改字符串，只需创建一个新字符串即可

##### 连接字符串

可以使用`+`运算符来连接两个字符串：

```kotlin
val str1 = "ab"
val str2 = "cde"
val result = str1 + str2 // "abcde"
```

当我们连接两个字符串时，会创建一个新字符串（因为字符串是**不可变的**）

如果字符串连接的是数字，则字符串必须在开头（第一个操作数）

```kotlin
val str = "abc" + 10 + true
println(str) // abc10true

val str = 10 + "abc" // an error here!
```

##### 比较字符串

比较两个字符串使用`==`（等于）和`!=`（不等于）

```kotlin
val first = "first"
val second = "second"
var str = "first"

println(first == str)  // true
println(first == second) // false
println(first != second) // true
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

## 算数运算

`Kolin` 中所有的运算符都有以下原则: 

### 二元运算符

`Kotlin` 有5个算数运算符

- 加 (addition) `+`
- 减 (subtraction) `-`
- 乘 (multiplication) `*`
- 除法 (division) `/`
- 取余 (remainder) `%` (modulus)

```kotlin
println(13 + 25) // prints 38

println(70 - 30) // prints 40

println(21 * 3)  // prints 63
```

除法 `/` 

```kotlin
// 参考类型推算
println(10 / 3)  	 // prints 3
println(10 / 3.0)  // prints 3.333333333
```

取余 `%` 得到除法的余数

```kotlin
println(10 % 3) // prints 1, because 10 divided by 3 leaves a remainder of 1
```

### 一元运算符

- 在一元加号表示为正值。它是一个可选的运算符，因此最好不要在实践中使用它

```java
println(+5) // prints 5
```

- 一元减号负值或表达式

```java
println(-8)  // prints -8
println(-(100 + 4)) // prints -104
```

它们都比**乘法**和**除法**运算符具有更高的优先级

### assignment 运算符

`assignment operations` （赋值算术运算符）是算术运算符和赋值运算符的组合

- `+=` 先加后赋值: **A += B** equals **A = A + B**
- `-=` 先减后赋值: **A -= B** equals **A = A - B**
- `*=` 先乘后赋值: **A \*= B** equals **A = A \* B**
- `/=` 先除后赋值: **A /= B** equals **A = A / B**
- `%=` 先取余后赋值: **A%= B** equals **A = A% B**

### 自增、自减

`++` 自增：执行+1

`--` 自减：执行-1

```kotlin
var num = 3
num++  // 4, increment
num--  // 3, decrement
```

自增运算符和自减运算符都有两种非常重要的区别形式：前缀和后缀

#### 前缀形式

前缀形式在变量使用之前改变其值

```kotlin
var a = 10
val b = ++a
println(a)  // a = 11
println(b)  // b = 11
```

这里，首先 `a` 的值自增1，然后再将 `a` 的值赋值给 `b` ，所以这里 `a` 和 `b` 都是 `11`

#### 后缀形式

后缀形式在变量使用之后改变其值

```kotlin
var a = 10
val b = a++
println(a)  // a = 11
println(b)  // b = 10
```

这里，首先将 `a` 的值赋值给 `b` ，然后 `a` 的值自增1，所以这里 `a=11`  `b=10`

#### 优先级

优先级从高到低

1. 括号
2. 后缀自增/自减
3. 一元正负号，前缀自增/自减
4. 乘法，除法和模数
5. 加减
6. 赋值

## 逻辑运算符

`Kotlin` 有4个逻辑运算符： **NOT**, **AND**, **OR** and **XOR** 

- **NOT(!)** 是一元操作符，对 `Boolean` 类型的值取反

```kotlin
val f = false // f is false
val t = !f    // t is true
```

- **AND(&&)** 是二元运算符， 如果两个操作数都是 `true` 则结果是 `true`，其他情况都是 `false`

```kotlin
val b1 = false && false // false
val b2 = false && true // false
val b3 = true && false // false
val b4 = true && true  // true 
```

- **OR(||)** 是二元运算符， 只要一个操作数是 `true` 则结果是 `true`, 其他情况都是 `false`

```kotlin
val b1 = false || false // false
val b2 = false || true  // true
val b3 = true || false  // true
val b4 = true || true   // true
```

- **XOR** (xor) 是二元运算符， 如果两个操作数不同则结果是 `true` ，其他情况都是 `false`

```kotlin
val b1 = false xor false // false
val b2 = false xor true  // true
val b3 = true xor false  // true
val b4 = true xor true   // false
```

### 逻辑运算符的优先级

优先级从高到低

- `!` 非
- `xor` 异或
- `&&` 与
- `||` 或

## 关系运算符

关系运算符的优先级低于算数运算符

关系运算符的优先级高于逻辑运算符

`Kotlin` 提供了六个关系运算符来比较数字：

- `==` （等于）
- `!=` （不等于）
- `>` （比...更棒）
- `>=` （大于或等于）
- `<` （少于）
- `<=` （小于或等于）

## 类型推断

推断类型时，其结果类型是表达式中类型最大的那个类型

下图是类型转换的方向

![type-inference](./img/type-inference.png)



```kotlin
val num: Int = 100
val longNum: Long = 1000
val result = num + longNum // 1100, Long
```

### Short and Byte 类型

只有 `Short` 和 `Byte` 类型的表达式结果类型是 `Int` 

```kotlin
val hundred: Short = 100
val five: Byte = 5
val zero = hundred % five // 0, Int
```

如果想两个 `Short` 类型的结果类型也是 `Short` 类型则必须手动转换

```kotlin
val one: Short = 1
val five: Short = 5
val six = (one + five).toShort() // 6, Short
```

### 表达式结果类型

在不同数字类型的表达式中，结果类型按照以下方法判断：

1. 如果两个都是 `Double`, 则结果类型是 `Double`.
2. 否则，如果任何一个操作数的类型是 `Float`, 则结果类型是 `Float`.
3. 否则，如果任何一个操作数的类型是 `Long`, 则结果类型是 `Long`.
4. 其他情况结果类型是 `Int`.

## 函数

**在 `Kotlin` 中，所有函数都有返回值**

```kotlin
val result = println("text")
println(result) // kotlin.Unit
```

这里返回一个特殊的对象 `Unit`, 意思是：没有结果

## 输入输出

### 标准输出

`标准输出` 是程序可以给他发送一些信息，默认情况下`标准输出`在屏幕上显示信息，但是可以将其重定向到文件

`Kotlin` 有2个函数可将数据发送到标准输出：`println`和`print`

`println` 在屏幕上显示字符串并换行

`print` 在屏幕上显示字符串并将光标放在其后

### 标准输入

`标准输入` 是一个传入程序的数据流

默认情况下，标准输入从键盘获取数据，但是也可以从文件获取数据

#### 使用 readLine 函数

`readLine` 可以从标准输入中读取数据，它以字符串形式读取整行

```kotlin
// !! 两个感叹号？
val line = readLine()!!
```

#### 使用 `Java` 的 `scanner` 

`Scanner` 可以从标准输入读取不同类型的值（字符串、数字等）

```kotlin
import java.util.Scanner

val scanner = Scanner(System.`in`)
val line = scanner.nextLine() // read a whole line, i.e. "Hello, Kotlin"
val num = scanner.nextInt()   // read a number, i.e. 123
val string = scanner.next()   // read a string, i.e. "Hello" 读取一个单词
```

**System.\`in\`** 是标准输入流对象。`scanner`包装它作为内部数据源并提供一组方便的方法

## 标准库

### 数学库

`Kotlin`两个数学库可供使用：

- `Math` 从Java继承（仅适用于Java平台）
- `kotlin.math` 专为Kotlin开发的（自Kotlin 1.2起，它为Java平台和JS提供通用数学）

这两个库都有相似的数学函数，工作方式也相同

#### 函数列表

常用函数如下：

- **Math.abs(...)**返回其参数的绝对值
- **Math.sqrt(...)**返回其参数的平方根
- **Math.cbrt(...)**返回其参数的立方根
- **Math.pow(...，...)**将第一个参数的值提高为第二个参数的幂
- **Math.log(...)**返回其参数的自然对数
- **Math.min(...，...)**返回两个参数的较小值
- **Math.max(...，...)**返回两个参数中的较大者
- **Math.toRadians(...)**将以度为单位的角度转换为以弧度为单位(近似)的角度；
- **Math.sin(...)**返回以弧度为单位的给定角度的三角正弦值
- **Math.cos(...)**返回以弧度为单位的给定角度的三角余弦值
- **Math.tan(...)**返回以弧度为单位的给定角度的三角正切值
- **Math.random()**返回一个带正号的双精度值，大于或等于0.0且小于1.0
- **Math.floor(...)**返回小于或等参数的最大整数（返回类型是 Double）
- **Math.ceil(...)**返回大于或等于参数的整数（返回类型是 Double）
- **Math.round(...)**返回最接近参数的整数（根据算术规则舍入）

常量:

- **Math.PI** 数学 `π` 常量
- **Math.E** 自然对数的基数




































