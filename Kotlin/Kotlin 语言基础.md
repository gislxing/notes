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

##### 字符串模板

字符串模板可以将变量的值插入到文本中

1. 变量模板

   要将变量的值添加到字符串中，在变量名称前写一个美元符号: `$变量`

   ```kotlin
   val city = "Paris"
   val temp = "24"
   
   println("Now, the temperature in $city is $temp degrees Celsius.")
   // 和下面代码输出一样
   println("Now, the temperature in " + city + " is " + temp + " degrees Celsius.")
   ```

2. 表达式模板

   可以使用字符串模板将任意表达式的结果放入字符串中: `${表达式}` 

   ```kotlin
   val language = "Kotlin"
   println("$language has ${language.length} letters in the name")
   ```

##### 获取子字符串

1. 截取字符串一部分

   `substring(startIndex, lastIndex)` 截取字符串一部分(不包含 `lastIndex` 位置的字符 )

   如果省略 `lastIndex` 则表示从 `startIndex` 开始到字符串末尾

   ```kotlin
   val greeting = "Hello"
   println(greeting.substring(0, 3)) // "Hel"
   println(greeting.substring(1, 3)) // "el"
   println(greeting.substring(2))    // "llo"
   println(greeting.substring(4, 5)) // "o"
   ```

    `substringAfter`和`substringBefore`也可获得部分字符串，接受分隔字符作为参数，返回指定的字符首次出现位置的后面或者前面的字符串，如果分割字符串没有找到则返回整个字符串

   ```kotlin
   // greeting = "Hello"
   println(greeting.substringAfter('l'))  // "lo"
   println(greeting.substringBefore("o")) // "Hell"
   println(greeting.substringBefore('z')) // "Hello"
   ```

2. 替换部分字符串

   `replace` 函数可用第二个参数替换第一个参数

   ```kotlin
   val example = "Good morning..."
   println(example.replace("morning", "bye")) // "Good bye..."
   println(example.replace('.', '!'))         // "Good morning!!!"
   
   ```

   如果只需要替换参数的**第一个匹配**项，请使用`replaceFirst`

   ```kotlin
   val example = "one one two three"
   println(example.replaceFirst("one", "two")) // "two one two three"
   ```

3. 重复字符串

   `repeat` 函数将字符串重复字符串多次

   ```kotlin
   print("Hello".repeat(4)) // HelloHelloHelloHello
   ```

##### 处理字符串

字符串像数组一样可以迭代

1. 字符串和数组相互转换

   ```kotlin
   val chars = charArrayOf('A', 'B', 'C', 'D', 'E', 'F')
   
   val stringFromChars = String(chars) // "ABCDEF"
   
   val charsFromString = stringFromChars.toCharArray() // { 'A', 'B', 'C', 'D', 'E', 'F' }
   
   val theSameString = String(charsFromString) // "ABCDEF"
   
   val text = "Hello world"
   val parts: Array<String> = text.split(" ").toTypedArray() // {"Hello", "world"}
   ```

2. 遍历字符串

   可以使用 `loop` 遍历字符串中的字符

   ```kotlin
val scientistName = "Isaac Newton"
   
   for (i in 0 until scientistName.length) {
       print("${scientistName[i]} ") // print the current character
   }
   
   for (ch in scientistName) {
   		print("$ch ") // print the current character
}
   ```

   通过索引迭代数组的示例：
   
   ```kotlin
   val rainbow = "ROYGCBV"
   
   for (index in rainbow.indices){
       println("${index+1}: ${rainbow[index]}")
   }
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

## 数据类型和变量

### 类型推断

推断类型时，其结果类型是表达式中类型最大的那个类型

下图是类型转换的方向

![type-inference](./img/type-inference.png)



```kotlin
val num: Int = 100
val longNum: Long = 1000
val result = num + longNum // 1100, Long
```

#### Short and Byte 类型

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

#### 表达式结果类型

在不同数字类型的表达式中，结果类型按照以下方法判断：

1. 如果两个都是 `Double`, 则结果类型是 `Double`.
2. 否则，如果任何一个操作数的类型是 `Float`, 则结果类型是 `Float`.
3. 否则，如果任何一个操作数的类型是 `Long`, 则结果类型是 `Long`.
4. 其他情况结果类型是 `Int`.

### 范围 range

`Kotlin` 提供了一种使用**range**表示范围

```kotlin
val within = c in a..b
```

`a..b` 表示从`a`到`b`数字范围（包括两个边界），`in`是一个特殊的关键字，用于检查值是否在范围内

如果 `c` 在范围内则 `within=true` ，否则 `winthin=false`

```kotlin
println(5 in 5..15)  // true
println(12 in 5..15) // true
println(15 in 5..15) // true
println(20 in 5..15) // false
```

如果检测一个值不在一个范围内，则只需要使用 `!in`

```kotlin
val notWithin = 100 !in 10..99 // true
```

可以为变量分配范围，并在以后使用

```kotlin
val range = 100..200
println(300 in range) // false
```

### 对象 Objects

一切都是对象

在 `Kotlin` 中，每个变量和值都是一个对象，例如，整数5和字符串 "high" 是对象

#### 复制对象引用 Copying by reference

`Kotlin` 中变量和值仅仅只是指向对象：如果创建一个变量并为其赋值一个对象，则另一个变量也可以指向同一对象

`=`符号不会复制对象本身，而只会复制对其的引用

#### 状态和行为

对象拥有状态和行为

属性(property)：在 `Kotlin` 中，允许你访问对象的状态，这里的状态就是属性。只需要在对象的后面加上点(`.`)和属性名称就可访问

成员函数(member functions)：绑定到具体类型上的函数，这些被绑定的函数就代表了对象的行为

### 数组 array

数组是同一类型元素的集合，所有元素按顺序存储在内存中

创建数组时将确定要存储的元素的数量，并且无法更改。但是，您可以随时修改存储的元素

<img src="./img/array-desc.svg" style="zoom: 33%;" />

#### 创建具有指定元素的数组

`Kotlin` 提供了许多类型的数组：`IntArray`, `LongArray`, `DoubleArray`, `FloatArray`, `CharArray`, `ShortArray`, `ByteArray`, `BooleanArray` 存储对应类型的元素，**注意这里没有 `StringArray`**

要创建指定类型的数组，我们需要调用一个特殊函数并传递所有元素以将它们存储在一起：

- `intArrayOf` 创建 `IntArray`
- `charArrayOf` 创建 `CharArray` 
- `doubleArrayOf` 创建 `DoubleArray` 

```kotlin
val numbers = intArrayOf(1, 2, 3, 4, 5) // It stores 5 elements of the Int type
println(numbers.joinToString()) // 1, 2, 3, 4, 5

val characters = charArrayOf('K', 't', 'l') // It stores 3 elements of the Char type
println(characters.joinToString()) // K, t, l

val doubles = doubleArrayOf(1.25, 0.17, 0.4) // It stores 3 elements of the Double type
println(doubles.joinToString()) // 1.15, 0.17, 0.4
```

#### 创建具有指定大小的数组

要创建具有指定大小的数组

```kotlin
val numbers = IntArray(5) // an array for 5 integer numbers
println(numbers.joinToString())

val doubles = DoubleArray(7) // an array for 7 doubles
println(doubles.joinToString())
```

这些具有预定义大小的数组由相应类型的默认值填充（数字类型为零）

#### 数组的大小

数组始终具有大小，即元素数。要获得它，我们需要获取`size`属性的值。它是Int类型的数字

```kotlin
val numbers = intArrayOf(1, 2, 3, 4, 5)
println(numbers.size) // 5 
```

创建数组后，无法更改其大小

#### 访问元素

通过索引设置值：

```kotlin
array[index] = elem
```

通过索引获取值：

```kotlin
val elem = array[index]
```

`Kotlin` 提供了几种方便的方法来访问数组的第一个和最后一个元素以及最后一个索引：

```kotlin
val strings = arrayOf("abc", "cdf", "efg")
println(strings.first()) // "abc"
println(strings.last()) // "efg"
println(strings.lastIndex) // 2
```

使用这种方法可以使您的代码更具可读性，并防止访问不存在的索引

#### 比较数组

要比较两个数组，请调用`contentEquals()`函数，并传递另一个作为参数。当两个数组以相同的顺序包含完全相同的元素则返回`true`，否则返回`false`：

```kotlin
val numbers1 = intArrayOf(1, 2, 3, 4)
val numbers2 = intArrayOf(1, 2, 3, 4)
val numbers3 = intArrayOf(1, 2, 3)

println(numbers1.contentEquals(numbers2)) // true
println(numbers1.contentEquals(numbers3)) // false
```

#### indices

`数组名.indices` 获得数组的索引范围

```kotlin
val arr = intArrayOf(1, 2, 3, 4)
for (i in arr.indices) {
  // 打印数组的每个索引
  println(i)
}
```

`数组名.indices - 整数` 表示跳过 `整数` 表示的索引

```kotlin
for (i in arr.indices - 2) {
  // 这里跳过了索引2，只打印 0 1 3
  println(i)
}
```

### 正则表达式

#### 创建一个正则表达式

`Kotlin` 有两种方式创建正则表达式

1. 创建一个`String`实例，然后调用 `toRegex()`，这将使该字符串产生一个正则表达式：

```kotlin
val string = "cat" // creating the "cat" string
val regex = string.toRegex() // creating the "cat" regex
```

2. 调用`Regex`构造函数

```kotlin
val regex = Regex("cat") // creating a "cat" regex
```

#### 简单匹配

```kotlin
String.matches(regex: String): Boolean
```

用于查找**完全**匹配，即整个字符串必须与模式匹配

```kotlin
val regex = Regex("cat") // creating the "cat" regex
    
val stringCat = "cat"
val stringDog = "dog"
val stringCats = "cats"

println(stringCat.matches(regex))   // true
println(stringDog.matches(regex))   // false
println(stringCats.matches(regex))  // false
```

#### 点字符(.)

点`.`匹配任何单个字符，包括字母，数字，空格等。它不能匹配的唯一字符是换行符`\n`

```kotlin
val regex = Regex("cat.") // creating the "cat." regex

val stringCat = "cat."
val stringEmotionalCat = "cat!"
val stringCatN = "cat\n"

println(stringCat.matches(regex))   // true
println(stringEmotionalCat.matches(regex))   // true
println(stringCatN.matches(regex))  //false
```

#### 问号(?)

问号`?`是一个特殊字符，表示**可选项**。它的意思是: 前面的字符或什么都没有

```kotlin
val regex = Regex("cats?") // creating the "cats?" regex

val stringCat = "cat"
val stringManyCats = "cats"

println(stringCat.matches(regex))   // true
println(stringManyCats.matches(regex))   // true
```

### BigInteger

Java类库提供了一个`BigInteger`用于处理非常大的数字（正数和负数）的类。因此，当您使用 `Kotlin/JVM` 环境时，可以使用`BigInteger` 

`BigInteger`类是**不可变的**，这意味着函数和类返回新实例

`BigInteger`除非绝对必要，否则不要使用。使用它总是会影响性能。`BigInteger`操作比内置整数类型的操作慢

#### 创建BigInteger对象

`BigInteger`属于`java.math`包，我们通过编写以下语句将其导入：

```kotlin
import java.math.BigInteger

val number1 = BigInteger("62957291795228763406253098")
val number2 = BigInteger.valueOf(1000000000)
val number3 = 1234.toBigInteger()
```

此外，该类具有几个有用的常量。使用特定常量比创建额外对象快一点：

```kotlin
val zero = BigInteger.ZERO // 0
val one = BigInteger.ONE   // 1
val ten = BigInteger.TEN   // 10
```

### null 和 non-null 类型

`NullPointerException` 类型问题的处理

`Kotlin` 中只有以下三种引发 `NullPointerException` 类型异常

1. 明确抛出：`throw NullPointerException()`
2. `!!` 语法
3. 错误的初始化，例如：构造函数和父类构造函数

`Kotlin` 中的每个引用都可以为空或不为空

```kotlin
// 默认下面的定义方式是：不为 null 类型定义
var name: String = null
```

这个将不能编译，因为我们声明了一个 `non-null` 变量

```kotlin
var name: String? = null
```

这里的 `?` 表示变量 `name` 可以是 `null`

#### 访问 null 变量

假设下面的定义

```kotlin
var name: String? = null
print(name.length)
```

这段代码不能被编译

有2中方法可以访问 `null变量` 

1. ```kotlin
   if (name != null) {
       print(name.length)
   }
   ```

2. 使用安全调用 `?.`

   ```kotlin
   print(name?.length)
   ```

### 类型系统

#### Any

`Kotlin` 中， `Any` 是所有非空类型的根类（root ），这意味着所有非空类型是 `Any` 的子类

`Kotlin` 保证 `Any` 类型的子类型永远不能为 `null`，所以在 `Kotlin` 中 `null` 检查是无用的

`Any?`（可以为null） 类型是 `Any` 类型的父类

<img src="./img/any2.png" style="zoom: 33%;" />



`null类型` 是 `non-null类型` 的父类，例如，`Number` 是 `Number?` 的子类

#### Unit

`Unit` 类型可以作为没有返回值的函数的返回类型

```kotlin
fun logCurrentState(): Unit { 
    println("Current state of a program: $state")
}
```

如果一个函数未指定返回值，则编译器会为其返回 `Unit`

```kotlin
fun updateState(state: State) { 
    logCurrentState()
    this.state = state
    logCurrentState()
}

val result: Unit = logCurrentState()
```

#### Nothing

`Nothing` 在 `Kotlin` 继承树的最底层

`Nothing` 是一个没有实例的类型

#### `Kotlin` 继承关系图

![](/Users/giszbs/Documents/notes/Kotlin/img/type-all.png)

## 流程控制

### if 表达式

**`Kotlin` 中 `if` 是表达式不是语句**

#### if-case

如果表达式是 `true` 则执行语句块`{}`中的代码，否则跳过

```kotlin
if (expression) {
    // body: do something
}
```

#### if-else-case

如果表达式为 `true` 则执行第一个代码块，否则执行第二个代码块

```kotlin
if (expression) {    
    // do something
} else {
    // do something else
}
```

#### if-else-if-case

```kotlin
if (expression0) {
    // do something
} else if (expression1) {
    // do something else 1
		// ...
} else if (expressionN) {
    // do something else N
}
```

#### if 表达式

与其他语言（例如`Java`，`Python`，`C＃`）不同，`Kotlin`中的`if`是表达式而不是语句，他可以返回一个计算结果，注意，这里的**返回结果必须是代码块的最后一个表达式**

**如果要将`if`用作表达式，则它必须有一个`else`分支**

```kotlin
// 计算最大值，打印然后将该值赋值给 max
val max = if (a > b) {
    println("Choose a")
    a
} else {
    println("Choose b")
    b
}
```

如果语句块仅包含一个语句，则可以省略花括号：

```kotlin
val max = if (a > b) a else b
```

### When 表达式

`when`表达式根据变量的值执行不同的操作，可以代替多分支**if表达式**

```kotlin
import java.util.*

fun main(args: Array<String>){
    val scanner = Scanner(System.`in`)

    val a = scanner.nextInt()
    val op = scanner.next()
    val b = scanner.nextInt()

    when (op) {
        "+" -> println(a + b)
        "-" -> println(a - b)
        "*" -> println(a * b)
        else -> println("Unknown operator")
    }
}
```

如果需要以同样的方式处理相同的情况，在一个分支中使用逗号组合

```kotlin
when (op) {
    "+", "plus" -> {
        val sum = a + b
        println(sum)
    }
    "-", "minus" -> {
        val diff = a - b
        println(diff)
    }
    "*", "times" -> {
        val product = a * b
        println(product)
    }
    else -> println("Unknown operator")
}
```

#### 作为表达式

`when`可以用作返回结果的表达式，此时每个分支都将返回值，并且必须有 `else` 分支

```kotlin
val result = when (op) {
    "+" -> a + b
    "-" -> a - b
    "*" -> a * b
    else -> "Unknown operator"
}
println(result)
```

这里的**返回结果必须是代码块的最后一个表达式** ，即是分支的最后一行

检测是否属于范围

```kotlin
when (n) {
    0 -> println("n is zero")
    in 1..10 -> println("n is between 1 and 10 (inclusive)")
    in 25..30 -> println("n is between 25 and 30 (inclusive)")
    else -> println("n is outside a range")
}
```

#### 没有参数的 when

使用不带参数的 `when` 结构，此时每个分支的条件都是一个简单的 `boolean 表达式` ，当分支的条件为 `true` 时执行该分支，如果有多个分支的条件都是 `true` 则只执行第一个分支

```kotlin
import java.util.*

fun main(args: Array<String>){
    val scanner = Scanner(System.`in`)
    val n = scanner.nextInt()
    
    when {
        n == 0 -> println("n is zero")
        n in 100..200 -> println("n is between 100 and 200")
        n > 300 -> println("n is greater than 300")
        n < 0 -> println("n is negative")
        // else-branch is optional here
    }
}
```

### 重复代码块

#### `repeat` 循环

最简单的循环就是使用 `repeat(n)` 并在大括号中`{...}` 写上需要循环的代码，`n` 是整数：表示循环的次数，如果`n`小于或等于零，则将忽略循环

```kotlin
repeat(n) {
    // statements
}
```

例如

```kotlin
// 循环打印3行 Hello
fun main(args: Array<String>) {
    repeat(3) {
        println("Hello")
    }
}
```

### for 循环

`Kotlin` 提供了`for`循环遍历范围、数组和集合

```kotlin
for (element in source) {
    // body of loop
}

for (element in array) {
    // body of loop
}
```

#### 遍历 range

```kotlin
for (i in 1..4) {
    println(i)    
}

for (ch in 'a'..'c') {
    println(ch)
}
```

#### 反向循环

```kotlin
for (i in 4 downTo 1) {
    println(i)
}
```

#### 排除上限

使用 `until` 排除上限

```kotlin
// 循环打印数字1到3
for (i in 1 until 4) {
    println(i)
}
```

#### 使用步进 step

如果没有指定 `step`，则 `step` 是 `1`

```kotlin
for (i in 1..7 step 2) {
    println(i)
}
```

反向循环也可使用 `step`

```kotlin
for (i in 7 downTo 1 step 2) {
    println(i)
}
```

### while 循环









## 函数

**在 `Kotlin` 中，所有函数都有返回值**

```kotlin
val result = println("text")
println(result) // kotlin.Unit
```

这里返回一个特殊的对象 `Unit`, 意思是：没有结果

### 声明函数

函数是将指令组合在一起以执行操作的序列，即它是一种子程序。函数具有名称

```kotlin
fun functionName(p1: Type1, p2: Type2, ...): ReturnType {
    // body
    return result
}
```

函数具有以下组件：

- 遵循与变量名称相同的规则和建议的名称
- 括号中代表输入数据的参数列表。每个参数都有一个名称和冒号`:`分隔的类型。所有参数用逗号分隔`,`
- 返回值的类型（可选）
- 包含要执行的语句和表达式的主体
- 关键字`return`后跟的结果（也是可选的）

有两种方法可以声明不返回任何值的函数（省略返回值类型，则默认返回值类型是 `Unit`）：

- 不指定返回类型（推荐使用）

```kotlin
/**
 * The function prints the values of a and b
 */
fun printAB(a: Int, b: Int) {
    println(a)
    println(b)
}
```

- 指定特殊类型`Unit`作为返回类型：

```kotlin
/**
 * The function prints the sum of a and b
 */
fun printSum(a: Int, b: Int): Unit {
    println(a + b)
}
```

#### 单个表达式函数

如果函数返回单个表达式，则可以省略大括号，此时返回值类型可以省略（`Kotlin` 会推断）

```kotlin
fun sum(a: Int, b: Int): Int = a + b
fun sum(a: Int, b: Int) = a + b // Int

fun sayHello(): Unit = println("Hello")
fun sayHello() = println("Hello") // Unit

fun isPositive(number: Int): Boolean = number > 0
fun isPositive(number: Int) = number > 0 // Boolean
```

### 默认参数

#### 默认参数函数

`Kotlin` 可以在声明中为函数的参数设置**默认值**。要调用此类函数，可以忽略具有默认值的参数，也可以通过所有参数以常规方式使用

```kotlin
fun printLine(line: String = "", end: String = "\n") = print("$line$end")

fun main(args: Array<String>) {
    printLine("Hello, Kotlin", "!!!") // prints "Hello, Kotlin!!!"
    printLine("Kotlin") // prints "Kotlin" with an ending
    printLine() // prints an empty line like println()
}
```

我们只需要在类型后面跟 `=` 号并赋值即可

#### 混合使用常规参数和默认参数

可以在函数的声明中混合使用默认参数和常规参数

```kotlin
fun findMax(n1: Int, n2: Int, absolute: Boolean = false): Int {
    val v1: Int
    val v2: Int

    if (absolute) {
        v1 = Math.abs(n1)
        v2 = Math.abs(n2)
    } else {
        v1 = n1
        v2 = n2
    }

    return if (v1 > v2) n1 else n2
}

fun main() {
    println(findMax(11, 15)) // 15
    println(findMax(11, 15, true)) // 15
    println(findMax(-4, -9)) // -4
    println(findMax(-4, -9, true)) // -9
}
```

#### 默认参数类型

默认参数可以是：常数、变量、另一个命名参数或者函数

```kotlin
fun sum2(a: Int, b: Int = a) = a + b
 
sum2(1)    // 1 + 1
sum2(2, 3) // 2 + 3
```

### 命名参数

调用具有参数的函数时，可以通过这些参数的名称传递参数

`命名参数` 必须在位置参数的后面，如果全部是`命名参数`则位置随便

```kotlin
fun calcEndDayAmount(startAmount: Int, ticketPrice: Int, soldTickets: Int) =
        startAmount + ticketPrice * soldTickets

// 位置参数方式调用
val amount = calcEndDayAmount(1000, 10, 500)  // 6000

// 命名参数方式调用
val amount = calcEndDayAmount(ticketPrice = 10, soldTickets = 500,startAmount = 1000)  // 6000

// 位置参数和命名参数混合调用
val amount = calcEndDayAmount(1000, ticketPrice = 10, soldTickets = 500)  // 6000
```

### 将函数用做对象

How to store a function as an object and to use it

只要函数的类型相同，则认为是相同的类型

`Kotlin` 中的一等公民条件如下：

1. 可以存储为变量
2. 可以被函数返回
3. 可以作为参数传递给函数
4. 不要依赖他们的名字
5. 可以在程序**运行时**创建

`Kotlin` 中函数是一等公民

#### 函数类型

`Kotlin` 内置支持函数类型

```kotlin
(parameters types) -> return value type
```

小括号中的参数类型是用逗号分隔的

```kotlin
fun sum(a: Int, b: Int): Int = a + b
// 上面函数的类型如下
// (Int, Int) -> Int
```

#### 函数引用

`Kotlin` 允许获取函数的引用

要获取一个对顶级函数的引用，只需要在函数名称前加上双引号(`::`)不需要需要小括号和参数，例如：`::sum` 将获得一个对 `(Int, Int) -> Int` 函数类型的引用

```kotlin
val sumObject = ::sum
```

`sumObject` 存储的是 `sum` 函数的引用，它具有 `sum` 函数的类型

还可以显示指定 `sumObject` 的类型

```kotlin
val sumObject: (Int, Int) -> Int = ::sum
```

这样就可以通过以下方式调用 `sum` 函数

```kotlin
sumObject(10, 20)
```

#### 返回其他函数引用

创建返回函数引用的函数

```kotlin
fun getRealGrade(x: Double) = x
fun getGradeWithPenalty(x: Double) = x - 1

fun getScoringFunction(isCheater: Boolean): (Double) -> Double {
    if (isCheater) {
        return ::getGradeWithPenalty
    }

    return ::getRealGrade
}
```

#### 函数引用作为参数传递

可以创建将其他函数作为参数的函数

```kotlin
fun applyAndSum(a: Int, b: Int, transformation: (Int) -> Int): Int {
    return transformation(a) + transformation(b)
}

fun square(x: Int) = x * x
applyAndSum(1, 2, ::square)  // returns 5 = 1 * 1 + 2 * 2
```

这里 `transformation` 参数接收一个 `(Int) -> Int` 类型的函数

```kotlin
fun isNotDot(c: Char): Boolean = c != '.'
val originalText = "I don't know... what to say..."
val textWithoutDots = originalText.filter(::isNotDot)
// "I don't know what to say"
```

### Lambda表达式

在运行时创建没有名字的函数

#### 没有名字的函数

`Kotlin` 中创建没有名字的函数：匿名函数、Lambda表达式

- 匿名函数：`fun(arguments): ReturnType { body }` 
- lambda表达式：`{ arguments -> body }` 

```kotlin
// 匿名函数
val mul1 = fun(a: Int, b: Int): Int {
    return a * b
}

// lambda表达式
val mul2 = { a: Int, b: Int -> a * b }

println(mul1(2, 3))  // prints "6"
println(mul2(2, 3))  // prints "6" too
```

不带参数的 `lambda表达式` 定义时不需要写箭头符号 `{ body }`

`lambda` 表达式只有一个参数时，可以使用 `it` 省略参数

```kotlin
val originalText = "I don't know... what to say..."
originalText.filter({ c -> c != '.' })
originalText.filter() { c -> c != '.' }
originalText.filter { c -> c != '.' }
originalText.filter { it != '.' }
```

如果 `lambda表达式` 有多行，则 `lambda表达式` 内的最后一行是其返回值

```kotlin
val textWithoutSmallDigits = originalText.filter {
    val isNotDigit = !it.isDigit()
    val stringRepresentation = it.toString()

    isNotDigit || stringRepresentation.toInt() >= 5
}
```

`lambda表达式` 也可以使用 `return` 返回，此时返回语句必须这样写：`return@标签名称` ，`标签名称`通常是传递 `lambda` 的函数名

```kotlin
val textWithoutSmallDigits = originalText.filter {
    if (!it.isDigit()) {
      	// 这里的 filter 是上面函数的名称
        return@filter true
    }
        
    it.toString().toInt() >= 5
}
```

#### 捕获变量

如果 `lambda` 使用在 `lambda表达式` 之外声明的变量，则表示 `lambda` **捕获**了该变量

在捕获值的情况下，`lambda`可以读取它。如果捕获了变量，则`lambda`和外部代码可以对其进行更改，并且这些更改将在`lambda`和外部代码中可见

```kotlin
var count = 0

val changeAndPrint = {
    ++count
    println(count)
}

println(count)    // 0
changeAndPrint()  // 1
count += 10
changeAndPrint()  // 12
println(count)    // 12
```

### main() 函数

`main()函数` 是程序入口函数

```kotlin
// 不带参数的 main 函数
fun main() {}

// 带参数的 main 函数
fun main(args: Array<String>) {}
```

`main` 函数也是可以带参数的，参数的名字约定为 `args`且类型是字符串数组

使用`main()`参数，我们将一些其他外部数据传输到程序本身

如果程序中**同时**包含**带参数和不带参数的 `main` 函数**，则程序会将带参数的`main`函数作为入口函数调用

#### 命令行参数

要将命令行参数发送到应用程序，需要一个预编译的程序

```shell
# 使用 jvm 在命令行运行程序
# args 是由空格分割的参数列表
$ java -jar filename.jar args
```

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

- **Math.abs(...)** 返回其参数的绝对值
- **Math.sqrt(...)** 返回其参数的平方根
- **Math.cbrt(...)** 返回其参数的立方根
- **Math.pow(...，...)** 将第一个参数的值提高为第二个参数的幂
- **Math.log(...)** 返回其参数的自然对数
- **Math.min(...，...)** 返回两个参数的较小值
- **Math.max(...，...)** 返回两个参数中的较大者
- **Math.toRadians(...)** 将以度为单位的角度转换为以弧度为单位(近似)的角度；
- **Math.sin(...)** 返回以弧度为单位的给定角度的三角正弦值
- **Math.cos(...)** 返回以弧度为单位的给定角度的三角余弦值
- **Math.tan(...)** 返回以弧度为单位的给定角度的三角正切值
- **Math.random()** 返回一个带正号的双精度值，大于或等于0.0且小于1.0
- **Math.floor(...)** 返回小于或等参数的最大整数（返回类型是 Double）
- **Math.ceil(...)** 返回大于或等于参数的整数（返回类型是 Double）
- **Math.round(...)** 返回最接近参数的整数（根据算术规则舍入）

常量:

- **Math.PI** 数学 `π` 常量
- **Math.E** 自然对数的基数




































