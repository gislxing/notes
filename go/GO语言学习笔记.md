# GO语言学习笔记

## 1. 入门

### 1.1 Hello World

　　Go语言的代码通过包（package）组织，包类似于其它语言里的库（libraries）或者模块（modules）。
一个包由位于单个目录下的一个或多个.go源代码文件组成, 目录定义包的作用。
每个源文件都以一条package声明语句开始, 表示该文件属于哪个包，紧跟着一系列导入（import）的包，之后是存储在这个文件里的程序语句。

> * main 包比较特殊,它定义了一个独立可执行的程序，而不是一个库。
> * 在 main 包里的 main 函数，它是整个程序执行时的入口。
> * 必须告诉编译器源文件需要哪些包，这就是 import 声明以及随后的 package 声明扮演的角色。
> * 必须恰当导入需要的包，缺少了必要的包或者导入了不需要的包，程序都无法编译通过。

hello world 示例代码如下：
```go
package main

import (
    "fmt"
)

func main()  {
    fmt.Println("Hello World")
}
```
**go** 命令编译一个或多个以 .go 结尾的源文件，链接库文件，并运行最终生成的可执行文件

```
go run helloworld.go
```

**go build** 命令生成一个名为 helloworld 的可执行的二进制文件

```
go build helloworld.go
```
### 1.2 命令行参数

`os`包以跨平台的方式，提供了一些与操作系统交互的函数和变量。程序的命令行参数可从`os`包的`Args`变量获取；`os`包外部使用`os.Args`访问该变量

`os.Args`变量是一个字符串（string）的*切片*（slice），切片是Go语言的基础概念

如果`s = os.Args`那么`s[i]`访问单个元素，用`s[m:n]`获取子序列，Go语言采用左闭右开形式, 即区间包括第一个索引元素，不包括最后一个，比如`s[m:n]`这个切片，`0 ≤ m ≤ n ≤ len(s)`包含`n-m`个元素，该切片表达式，产生从第m个元素到第n-1个元素的切片

`os.Args`的第一个元素，`os.Args[0]`是命令本身的名字；其它的元素则是程序启动时传给它的参数

如果省略切片表达式的m或n，会默认传入`0`或`len(s)`，因此切片`os.Args[1:]`表示获取所有程序启动时传递的参数

注释语句以`//`开头

以下是一个`echo`命令的简单实现例子：

```go
package main

import (
	"os"
	"fmt"
)

func main() {
	var s, sep string
	for i := 1; i < len(os.Args); i++ {
		s += sep + os.Args[i]
		sep = " "
	}
	fmt.Println(s)
}
```

`var`声明定义了两个`string`类型的变量`s`和`sep`。变量会在声明时直接初始化。如果变量没有显式初始化，则被隐式地赋予其类型的*零值*（zero value），数值类型是`0`，字符串类型是空字符串`""`

符号`:=`是*短变量声明*（short variable declaration）的一部分, 这是定义一个或多个变量并根据它们的初始值为这些变量赋予适当类型的语句

`i++`是语句，而不像C系的其它语言那样是表达式。所以`j = i++`非法，而且`++`和`--`都只能放在变量名后面

`echo`例子改进：

```go
package main

import (
    "fmt"
)

func main() {
    s, sep := "", ""
    for _, arg := range os.Args[1:] {
        s += sep + arg
        sep = " "
    }
    fmt.Println(s)
}
```

每次循环迭代，`range`产生一对值，索引以及在该索引处的元素值，这个例子不需要索引，但`range`的语法要求, 要处理元素, 必须处理索引。因为Go语言不允许使用无用的局部变量（local variables），这会导致编译错误，所以Go语言中这种情况的解决方法是用`空标识符`（blank identifier），即`_`（也就是下划线），空标识符可用于任何语法需要变量名但程序逻辑不需要的时候

声明一个变量有好几种方式，下面这些都等价：

```
// 只能用在函数内部，而不能用于包变量
s := ""

// 默认初始化零值机制，被初始化为""
var s string

// 用得少，除非同时声明多个变量
var s = ""

// 显式地标明变量的类型，当变量类型与初值类型相同时，类型冗余，但如果两者类型不同，变量类型就必须了
var s string = ""
```

如果连接涉及的数据量很大，这种方式代价高昂，高效的解决方案是使用`strings`包的`Join`函数：

```
func main() {
    fmt.Println(strings.Join(os.Args[1:], " "))
}
```

### 1.3 查找重复的行

`map`存储了键/值（key/value）的集合，对集合元素，提供常数时间的存、取或测试操作。键可以是任意类型，只要其值能用`==`运算符比较，最常见的例子是字符串；值则可以是任意类型

`bufio`包，它使处理输入和输出方便又高效。`Scanner`类型是该包最有用的特性之一，它读取输入并将其拆成行或单词

`fmt.Printf`函数对一些表达式产生格式化输出。该函数的首个参数是个格式字符串，指定后续参数被如何格式化。各个参数的格式取决于“转换字符”（conversion character），形式为百分号后跟一个字母。举个例子，`%d`表示以十进制形式打印一个整型操作数，而`%s`则表示把字符串型操作数的值展开。

`Printf`有一大堆这种转换，Go程序员称之为*动词（verb）*。下面的表格虽然远不是完整的规范，但展示了可用的很多特性：

```
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

下面是第一个`dup`的例子，该例子是从控制台获取输入，计算其中的重复行并输出

```go
package main

import (
   "bufio"
   "fmt"
   "os"
)

func main() {
   counts := make(map[string]int)
   input := bufio.NewReader(os.Stdin)
   var s string

   for true {
      fmt.Print("Please enter some input: ")
      // 读取输入的值，以回车作为读取的结束标志
      s, _ = input.ReadString('\n')

      if s == "end\n" {
         break
      }

      counts[s]++
   }

   for line, n := range counts {
      if n > 1 {
         fmt.Printf("%d\t%s\n", n, line)
      }
   }
}
```

下面是打开一个文件并打印重复行的第二个`dup`例子

```go
package main

import (
   "bufio"
   "fmt"
   "os"
)

func main() {
    counts := make(map[string]int)
	filename := "E:\\GitHub\\notes\\go\\go环境配置.txt"
	data, err := ioutil.ReadFile(filename)
	if err != nil {
		fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
	} else {
		for _, line := range strings.Split(string(data), "\n") {
			counts[line]++
		}
	}

	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%s\n", n, line)
		}
	}
}
```

### 1.4 GIF动画



### 1.5 获取URL

Go语言在net这个强大package的帮助下提供了一系列的package来做这件事情，使用这些包可以更简单地用网络收发信息，还可以建立更底层的网络连接，编写服务器程序。

下面是一个爬取网页源码的示例

```go
// 爬取指定网址的源码
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
)

func main() {
	url := "http://gopl-zh.b0.upaiyun.com/ch1/ch1-05.html"
	if !strings.HasPrefix(url, "http://") {
		url = "http://" + strings.Trim(url, " ")
	}
	response, err := http.Get(url)
	if err != nil {
		fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
		os.Exit(1)
	}
	_, err = io.Copy(os.Stdout, response.Body)
	response.Body.Close()
	if err != nil {
		fmt.Fprintf(os.Stderr, "fetch: read %s : %s\n", url, err)
		os.Exit(1)
	}
	fmt.Printf("response status : " + response.Status)
}
```

### 1.6 并发获取多个URL

`goroutine`是一种函数的并发执行方式，而`channel`是用来在`goroutine`之间进行参数传递。main函数本身也运行在一个`goroutine`中，而`go function`则表示创建一个新的`goroutine`，并在这个新的`goroutine`中执行这个函数。

`ioutil.Discard`输出流相当于一个`垃圾站`，可以将我们不用的数据写入

### 1.7 Web服务

首先看一个简单的服务器例子：

```go
// Server1 is a minimal "echo" server.
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", handler) // each request calls handler
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the request URL r.
func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

[浏览器访问](http://localhost:8000) 即可看到我们的第一个web服务

如果你的操作系统是`Mac OS X`或者`Linux`，那么在运行命令的末尾加上一个`&`符号，即可让程序简单地跑在后台

### 1.8 要点总结

**控制流：** 在本章我们只介绍了if控制和for，但是没有提到switch多路选择。这里是一个简单的switch的例子：

```
switch coinflip() {
case "heads":
    heads++
case "tails":
    tails++
default:
    fmt.Println("landed on edge!")
}

```

Go语言并不需要显式地在每一个case后写break，语言默认执行完case后的逻辑语句会自动退出。当然了，如果你想要相邻的几个case都执行同一逻辑的话，需要自己显式地写上一个fallthrough语句来覆盖这种默认行为。不过fallthrough语句在一般的程序中很少用到。

Go语言里的switch还可以不带操作对象；可以直接罗列多种条件，像其它语言里面的多个if else一样，下面是一个例子：

```
func Signum(x int) int {
    switch {
    case x > 0:
        return +1
    default:
        return 0
    case x < 0:
        return -1
    }
}

```

这种形式叫做无tag switch(tagless switch)；这和switch true是等价的。

像for和if控制语句一样，switch也可以紧跟一个简短的变量声明，一个自增表达式、赋值语句，或者一个函数调用

break和continue语句会改变控制流。和其它语言中的break和continue一样，break会中断当前的循环，并开始执行循环之后的内容，而continue会中跳过当前循环，并开始执行下一次循环。这两个语句除了可以控制for循环，还可以用来控制switch和select语句，continue会跳过内层的循环，如果我们想跳过的是更外层的循环的话，我们可以在相应的位置加上label，这样break和continue就可以根据我们的想法来continue和break任意循环。

**命名类型：** 类型声明使得我们可以很方便地给一个特殊类型一个名字：

```
type Point struct {
    X, Y int
}
var p Point
```

**指针：** Go语言提供了指针。指针是一种直接存储了变量的内存地址的数据类型。指针是可见的内存地址，&操作符可以返回一个变量的内存地址，并且*操作符可以获取指针指向的变量内容，但是在Go语言里没有指针运算。

**方法和接口：** 方法是和命名类型关联的一类函数。Go语言里比较特殊的是方法可以被关联到任意一种命名类型。接口是一种抽象类型，这种类型可以让我们以同样的方式来处理不同的固有类型，不用关心它们的具体实现，而只需要关注它们提供的方法。

**包（packages）：** Go语言提供了一些很好用的package，并且这些package是可以扩展的。Go语言社区已经创造并且分享了很多很多。所以Go语言编程大多数情况下就是用已有的package来写我们自己的代码。

在你开始写一个新程序之前，最好先去检查一下是不是已经有了现成的库可以帮助你更高效地完成这件事情。你可以在 <https://golang.org/pkg> 和 [https://godoc.org](https://godoc.org/) 中找到标准库和社区写的package。godoc这个工具可以让你直接在本地命令行阅读标准库的文档。比如下面这个例子。

```
$ go doc http.ListenAndServe
package http // import "net/http"
func ListenAndServe(addr string, handler Handler) error
    ListenAndServe listens on the TCP network address addr and then
    calls Serve with handler to handle requests on incoming connections.
...

```

**注释：**多行注释可以用 `/* ... */` 来包裹，单行注释`//`

## 2. 程序结构

### 2.1. 命名

Go语言命名规则：一个名字必须以一个字母（Unicode字母）或下划线开头，后面可以跟任意数量的字母、数字或下划线，并且区分大小写。

Go语言中总共有25个关键字

```
break      default       func     interface   select
case       defer         go       map         struct
chan       else          goto     package     switch
const      fallthrough   if       range       type
continue   for           import   return      var

```

此外，还有大约30多个预定义的名字，主要对应内建的常量、类型和函数。

```
内建常量: true false iota nil

内建类型: int int8 int16 int32 int64
          uint uint8 uint16 uint32 uint64 uintptr
          float32 float64 complex128 complex64
          bool byte rune string error

内建函数: make len cap new append copy close delete
          complex real imag
          panic recover

```

这些内部预先定义的名字并不是关键字，你可以再定义中重新使用它们。

如果一个名字是在函数内部定义，那么它就只在函数内部有效。如果是在函数外部定义，那么将在当前包的所有文件中都可以访问。如果一个名字是大写字母开头的可以被外部的包访问，小写字母开头的只能在包内访问。包本身的名字一般用小写字母。

Go语言程序员推荐使用 **驼峰式** 命名，当名字有几个单词组成时优先使用大小写分隔。

### 2.2. 声明

声明语句定义了程序的各种实体对象以及部分或全部的属性。`Go`语言主要有四种类型的声明语句：`var`、`const`、`type`和`func`，分别对应变量、常量、类型和函数实体对象的声明。

一个Go语言编写的程序对应一个或多个以`.go`为文件后缀名的源文件中。每个源文件以包的声明语句开始，说明该源文件是属于哪个包。包声明语句之后是`import`语句导入依赖的其它包，然后是包一级的类型、变量、常量、函数的声明语句，包一级的各种类型的声明语句的顺序无关紧要

在包一级声明语句声明的名字可在整个包对应的每个源文件中访问，而不是仅仅在其声明语句所在的源文件中访问。局部声明的名字就只能在函数内部很小的范围被访问

一个函数的声明由一个函数名字、参数列表（由函数的调用者提供参数变量的具体值）、一个可选的返回值列表和包含函数定义的函数体组成。如果函数没有返回值，那么返回值列表是省略的。执行函数从函数的第一个语句开始，依次顺序执行直到遇到`renturn`返回语句，如果没有返回语句则是执行到函数末尾，然后返回到函数调用者

### 2.3. 变量

var声明语句可以创建一个特定类型的变量，然后给变量附加一个名字，并且设置变量的初始值。变量声明的一般语法如下：

```
var 变量名字 类型 = 表达式
```

其中`类型`或`= 表达式`两个部分可以省略其中的一个。如果省略的是类型信息，那么将根据初始化表达式来推导变量的类型信息。如果初始化表达式被省略，那么将用零值初始化该变量

数值类型变量对应的零值是`0`，布尔类型变量对应的零值是`false`，字符串类型对应的零值是空字符串，接口或引用类型（包括slice、map、chan和函数）变量对应的零值是`nil`。数组或结构体等聚合类型对应的零值是每个元素或字段都是对应该类型的零值。`Go`语言中不存在未初始化的变量

在包级别声明的变量会在`main`入口函数执行前完成初始化，局部变量将在声明语句被执行到的时候完成初始

#### 2.3.1. 简短变量声明

简短变量声明以`名字 := 表达式`形式声明变量，变量的类型根据表达式来自动推导

简短变量声明只能用在`函数内部`，即局部变量

简短变量声明语句也可以用来声明和初始化一组变量：

```
i, j := 0, 1
```

请记住`:=`是一个变量声明语句，而“=‘是一个变量赋值操作

简短变量声明左边的变量可能并不是全部都是刚刚声明的。如果有一些已经在相同的词法域声明过了，那么简短变量声明语句对这些已经声明过的变量就只有赋值行为了

简短变量声明语句中**必须至少**要声明一个**新的变量**

#### 2.3.2. 指针

一个变量对应一个保存了变量对应类型值的内存空间

一个指针的值是另一个变量的地址。一个指针对应变量在内存中的存储位置。并不是每一个值都会有一个内存地址，但是对于每一个变量必然有对应的内存地址。通过指针，我们可以直接读或更新对应变量的值，而不需要知道该变量的名字

如果用`var x int`声明语句声明一个`x`变量，那么`&x`表达式（取`x`变量的内存地址，`&`是取地址操作符）将产生一个指向该整数变量的指针，指针对应的数据类型是`*int`，指针被称之为**指向int类型的指针**。

如果`p := &x`那么`*p`表达式对应p指针指向的变量的值。一般`*p`表达式读取指针指向的变量的值，这里为`int`类型的值，同时因为`*p`对应一个变量，所以该表达式也可以出现在赋值语句的左边，表示更新指针所指向的变量的值

任何类型的指针的零值都是nil。指针之间也是可以进行相等测试的，只有当它们指向同一个变量或全部是nil时才相等。

在`Go`语言中，可以返回函数中局部变量的地址

**指针特别有价值**的地方在于我们可以不用名字而访问一个变量

#### 2.3.3. new函数

另一个创建变量的方法是调用内建的`new`函数。表达式`new(T)`将创建一个`T`类型的匿名变量，初始化为`T`类型的零值，然后返回变量地址，返回的指针类型为`*T`。

每次调用`new`函数都是返回一个**新的变量的地址**

可能有特殊情况：如果两个类型都是空的，也就是说类型的大小是0，例如`struct{}`和 `[0]int`, 有可能有相同的地址

由于`new`只是一个预定义的函数，它并不是一个关键字，因此我们可以将`new`名字重新定义为别的类型

#### 2.3.4. 变量的生命周期

变量的生命周期指的是在程序运行期间变量有效存在的时间间隔。对于在包一级声明的变量来说，它们的生命周期和整个程序的运行周期是一致的。局部变量的声明周期则是动态的：从每次创建一个新变量的声明语句开始，直到该变量不再被引用为止，然后变量的存储空间可能被回收

编译器会**自动选择**在**栈**上还是在**堆**上分配**局部变量**的存储空间

**逃逸**是指局部变量变为包变量的过程，逃逸的变量需要额外分配内存，同时对性能的优化可能会产生细微的影响

### 2.4. 赋值

使用赋值语句可以更新一个变量的值，最简单的赋值语句是将要被赋值的变量放在=的左边，新值的表达式放在=的右边

数值变量也可以支持`++`递增和`--`递减语句。自增和自减是语句，而不是表达式，因此`x = i++`之类的表达式是错误的

#### 2.4.1. 元组赋值

元组赋值是另一种形式的赋值语句，它允许同时更新多个变量的值。在赋值之前，赋值语句右边的所有表达式将会**先进行求值**，然后**再统一更新**左边对应变量的值

有些表达式会产生多个值，比如调用一个有多个返回值的函数。当这样一个函数调用出现在元组赋值右边的表达式中时（右边不能再有其它表达式），左边变量的数目必须和右边一致。

用下划线空白标识符`_`来丢弃不需要的值

#### 2.4.2. 可赋值性

赋值语句是显式的赋值形式，但是程序中还有很多地方会发生隐式的赋值行为：函数调用会隐式地将调用参数的值赋值给函数的参数变量，一个返回语句将隐式地将返回操作的值赋值给结果变量，一个复合类型的字面量也会产生赋值行为

对于两个值是否可以用`==`或`!=`进行相等比较的能力也和可赋值能力有关系：对于任何类型的值的相等比较，第二个值必须是对第一个值类型对应的变量是可赋值的，反之依然。

### 2.5. 类型

变量或表达式的类型定义了对应存储值的属性特征

类型声明语句一般出现在包一级，因此如果新创建的类型名字的首字符大写，则在外部包也可以使用。

对于每一个类型`T`，都有一个对应的类型转换操作`T(x)`，用于将`x`转为`T`类型，如果`T`是指针类型，可能会需要用小括弧包装`T`，比如`(*int)(0)`

只有当两个类型的底层基础类型相同时，才允许这种转型操作，或者是两者都是指向相同底层结构的指针类型，这些转换只改变类型而不会影响值本身

命名类型还可以为该类型的值定义新的行为。这些行为表示为一组关联到该类型的函数集合，我们称为类型的方法集

### 2.6. 包和文件

























































