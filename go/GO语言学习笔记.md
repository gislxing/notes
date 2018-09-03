# Go语言学习笔记

## 安装

    1、到 https://golang.google.cn/ 下载最新的安装包
    2、下载后直接双击msi文件安装，默认安装在c:\go
    4、验证是否安装成功，在运行中输入 cmd 打开命令行工具，在提示符下输入 go
    5、设置工作空间 gopath 目录(Go语言开发的项目路径)
        Windows 设置如下，新建一个环境变量名称叫做GOPATH，值为你的工作目录，例如: GOPATH=e:\mygo
    6、用 go env 命令查看环境变量设置
## 类型

### 变量

`var`用于定义变量，且类型在变量名后。运行时变量会被自动初始化为`零值`。如果定义变量的同时指定变量的值，则可省略变量类型，由编译器推断类型。

```go
var x int
var y = true
```

可一次定义多个变量。在多变量赋值时，先计算出所有右边的值，然后再依次赋值。

```go
var x, y int
```

多行变量建议使用组的形式定义

```go
var (
	x, y int
	a, b = 1, "1"
)
```

使用`:=`简短定义变量`a := 100`，使用简短变量定义有以下限制：

```markdown
定义变量的同时需要初始化
不能写变量类型
只能在函数内部使用
```

### 命名

首字母大写的是共有的（可在包外访问），首字母小写为私有。

空标识符`_`，可在表达式左边使用，忽略某个值。

### 常量

`const`用于定义常量

```go
const PI = 3.1415
```

在常量组中，如果不指定类型和初始化，则与上面非空行常量相同

```go
const (
	x = 123
	y				// 与x常量相同
	a = "zbx"
	b				// 与a常量相同
)
```

`iota`自动标识符，可用户创建一组自增变量或常量。如果中断`iota`，必须显示恢复，且后续自增值按行序递增

```go
const (
	_ = iota	// 0
	a			// 1
	b = 200
	c = iota	// 3
)
```

### 基本数据类型

![基本数据类型](img\go_基本数据类型.png)

标准库`math`定义了数字类型的取值范围，`strconv`可在不同进制之间转换

#### 别名

`byte` alias for `uint8`

`rune` alias for `int32`

别名类型可直接赋值

```go
var a uint8 = 10
var b byte = a
```

### 引用类型

引用类型有三种`slice、map、channel`，引用类型必须用`make`函数创建才可直接使用

`new`可按指定类型长度分配零值内存，并返回指针

`make`创建引用类型，编译器会将`make`转换为引用类型专用的创建函数，以保证内存分配和初始化

### 类型转换

`GO`必须显式进行类型转换，混合类型表达式必须保证类型一致

```go
a := 10
b := byte(a)
c := a + int(b)		// 类型要一致
```

 ### 自定义类型

`type`定义用户自定义类型，并且只继承基础类型的操作符。

#### 未命名类型

未命名类型：数组、切片、字典、通道等与具体元素类型或者长度有关。`type`可为未命名类型指定名称，将其变为命名类型。

## 表达式

### 保留字

```go
break 		default 	func 	interface 	select
case 		defer 		go 		map 		struct
chan 		else 		goto 	package 	switch
const 		fallthrough if 		range 		type
continue 	for 		import 	return 		var
```

### 运算符
全部运算符、分隔符，以及其他符号。运算符结合律全部从左到右

```go
+  &   += &=   &&  == !=  ( )
-  |   -= |=   ||  <  <=  [ ]
*  ^   *= ^=   <-  >  >=  { }
/  <<  /= <<=  ++  =  :=  , ;
%  >>  %= >>=  --  !  ... . :
&^ &^=
```

#### 自增

自增`++`、自减`--`是语句，不能用于表达式，并且只能在变量的右边

```go
a := 1
a++		// ++a 错误写法，++只能在变量的右边
```

#### 指针

`内存地址`是内存地址中每个字节的唯一编号，`指针`用于存储`内存地址`的变量。

`&`取地址云算法用于获取对象地址

`*`指针运算符用于引用对象

指针类型只支持相等运算符，即两个指针指向同一个地址或者都为`nil`时相等

可使用`unsafe.Pointer`将指针转换为`uintptr`后再进行加减运算

指针使用`.`指向成员

```go
a := struct{
    x int
}

a.x = 100
p := &a
fmt.println(p.x)	// 100
```

### 流控制

#### if...else...

条件表达式必须是布尔类型，可省略括号

支持块局部变量或者函数

```go
if a := x + 1; a < 10 {
    println(a)
}

if init(); b < 10 {
    println(b)
}
```

#### switch

`switch`也支持块局部变量或函数，按照从上到下、从坐到右的顺序匹配`case`

```go
switch a {
	case init(); b:
    	println("b")
    case c:
    	println("c")
    case 4:
    	println("4")
    default:
    	println("def")
}
```

`case`语句中不能出现重复的常量

`case`语句块执行后隐式自动执行`break`终止，如果使用`fallthrough`关键字则后面的所有`case`语句块都会执行（不执行case语句条件表达式），`fallthrough`必须在`case`语句块的最后，中间可使用`break`终止后续代码执行

```go
	switch x := 1; x {
	case 0:
		println("0")
	case 1:
		println("1")
        
        if x > 10 {
            break;
        }
        
		fallthrough
	case 2:
		println("2")
	default:
		println("def")
	}
```

#### for

`for`同样支持语句块或者函数调用

```go
for i := 0; i < 10; i++{
    
}
```

```go
for i < 10 {
    
}
```

```go
for {
    // 死循环
}
```

使用`for...range`进行迭代，支持字符串、数组、slice、map、channel，返回索引和键值

```go
arr := [3]string{"1", "2", "3"}

for i, s := range arr{
    println(i, s)
}

for i := range arr{
    println(i)
}

for range arr{
    
}
```

#### goto

`goto`语句无条件地跳转到过程中指定的标签位置，继续执行标签下面的代码

```go
	for a:=0;a<5;a++{
        fmt.Println(a)
        if a>3{
            goto Loop
        }
    }
    Loop:
    fmt.Println("test")
```

#### break、continue

`break`终止循环

`continue`终止后续代码执行，进入下一轮循环

## 函数

`func`用于定义函数，不支持重载

具有相同签名（参数和返回值列表）的函数是同一类型函数

函数只能判定是否为`nil`

```go
func a() {}
a == nil
```

### 参数

调用函数时，必须按签名顺序传递指定类型和数量的参数，在参数列表中相邻同类型的参数可合并

函数的参数是`值拷贝传递`

```go
func test(x, y int, s string){
    
}
```

### 变参

`变参`其实是切片，只能接受一到多个同类型参数，且必须在列表尾部，由于切片是引用类型所以可在调用的方法中修改原值

将切片/数组作为变参时，则需进行展开操作

```go
func test(a ...int){
    for i := range a {
		a[i] += 10
	}
}

func main(){
    a := []int{1,2,3}
    test(a...)
 	fmt.Println(a)		// 结果：[11 12 13]
}
```

### 返回值

函数可返回多个值

接收参数时，使用`_`可忽略不想要的值

```go
func test(x int) (int, error){
    return 1, errors.New("xxxx")
}
```

#### 命名返回值

即对返回值命名，该命名返回值就是该函数的局部变量，并最后由`return`隐式返回

```go
func test(x, y int) (z int, err error){
    if y == 0{
        err = errors.New("xxxx")
        return
    }
    
    z = x / y
    return
}
```

### 匿名函数

匿名函数是没有定义函数名称的函数

可在普通函数内部定义匿名函数，可赋值给变量

```go
f := func(a int){
    println(a)
}
```

#### 闭包

`闭包`是在其词法上下文中引用了自由变量的函数

`闭包`通过指针引用环境变量，这将导致环境变量的生命周期延长

```go
func test() []func(){
    var s []func()
    for i := 0; i < 3; i++ {
        s = append(i, func(){
            println(i)
        })
    }
    
    return s
}

func main(){
    for _, f := range test() {
        f()		// 输出 3
    }
}
```

上面代码输出的结果是：3，这个是由于闭包通过指针引用的变量，解决办法是每次给出新的变量

### 延迟调用

`defer`关键字用于定义延迟调用的函数，`defer`定义的延迟调用时在当前方法结束前调用

多个`defer`的延迟调用按照`FILO`（先入后出）顺序执行

常用于处理：资源释放、解除锁定、错误处理等

```go
func test(){
    f, err := os.Open("./test.txt")
    defer f.Close()		// 函数退出前执行
    if err != nil {
        return
    }
}
```

`延迟调用`的函数参数在定义时被复制并缓存起来，即在定义延迟调用函数时其参数就已经固定，如果需要与延迟调用之外的变量保持同步，则可使用指针或者闭包

### 错误处理

官方推荐返回`error`状态，`error`一般作为最后一个返回参数

```go
func test() (error)
```

`error`实现如下

```go
type error interface {
	Error() string
}
```

自定义错误类型

```go
// 自定义错误类型
type DivError struct{
    x, y int
}

// 实现 error 接口方法
func (DivError) Error() string {
    return "dis by zero"
}

func div(x, y int) (int, error) {
    if y == 0 {
        return 0, DivError{x, y}
    }
    
    return z / y, nil
}
```

#### panic / recover

`panic`会立即中断当前程序流程，并执行延迟调用，如果扔出多个异常，则只有最后一个异常可被捕获

`recover`可捕获并返回`panic`扔出的异常（注意：`recover`必须在延迟调用的函数中才有效）

```go
	defer func() {					// 可捕获panic扔出的异常
		if err := recover(); err != nil {
			log.Fatal(err)
		}
	}()
	defer println(recover())		// 无法补货panic扔出的异常

	panic("xxxx")
```

可使用`runtime/debug.PrintStack()`打印完整调用堆栈信息

## 数据

### 字符串

字符串是不可变字节序列，其零值是空字符串`""`

**`**符号可定义不转义处理的字符串，并可跨行

```go
s := `i am bird\r\n
you are?`
```

字符串支持`==、!=、< 、>、+、+=`操作符

```go
s := "aa" +				// 跨行时，加法操作符必须在末尾
	"bb"
println(a == "aabb")	// true, 字符串比较运算
println(s > "ab")
```

允许以索引号访问字符串中的元素，返回的是对应的ASC码

```go
s := "abcd"
println(s[0])			// 97
println(string(s[0]))	// a
```

`string`与`[]byte`互转

```go
s := "hello"
b := []byte(s)
d := string(b[:])	// b 与 d 公用内存
f := string(b)		// b 与 f 使用不同内存
```

使用`+`拼接字符串时，每次都要重新分配内存，所以，在构建较大字符串时性能极差，此时应使用`strings.Join()`和` bytes.Buffer()`

### 数组

定义数组时，元素类型和长度都是数组组成部分，即只有元素类型相同且长度相同的数组才是同一类型

```go
a := [2]int
b := [3]int
c := [3]int

a == b		// false
b == c		// true
```

```go
a := [4]int{}				// 元素自动初始化为零值
b := [4]int{4,5}			// 部分提供初始值，未提供初始值的自动初始化为零值
c := [4]int{2, 2:10}		// 在索引2处指定值10
d := [...]int{1, 2, 3, 4}	// 编译器按照初始值长度确定数组长度
```

在定义多维度数组时，只有第一维度可使用`...`

`len()`和`cap()`返回数组长度，在多维数组中只返回第一维度长度

如果数字元素支持`==`和`!=`操作符，那么该数组也支持这两个操作符（此时将比较数组中每个元素）

```go
a := [2]string{1,2}
b := [2]string{1,1}
c := [2]string{1,1}

a == b		// false
b == c		// true
```

### 切片

`slice`切片是通过指针引用数组底层数据

数组和切片的定义的区别是`[]`中是否有数字，即是否定义长度

不支持比较操作，只支持是否`nil`判定

```go
slice1 := make([]int32, 5, 8)
slice2 := make([]int32, 9)
slice3 := []int32{}
slice4 := []int32{1, 2, 3, 4, 5}
```

`append`向切片对象末尾追加元素并返回新的切片对象，向`nil`切片追加数据时会为其分配地址数组内存

### 字典

字典是无序键值对集合，其中`key`必须是支持相等运算（==、!=）符的数据类型

```go
m := make(map[int]string)
```

访问不存在的键值时返回零值，不会发生错误

在迭代期间新增或者删除键值是安全的

```go
	m := make(map[int]int)

	for i := 0; i < 10; i++ {
		m[i] = i * 10
	}

	for k := range m {
		if k%2 == 0 {
			delete(m, k)
		}

		fmt.Println(m)
	}
```

### 结构

`struct`结构将不同类型数据组合成一个复合类型

```go
type user struct {
    name string
    age  int
}
```

只有结构的所有字段都支持比较操作时，那么该结构才支持比较操作

```go
	type user struct {
		name string
		age  int
	}

	a := user{
		name: "aaa",
		age:  10,
	}

	b := user{
		name: "aaa",
		age:  10,
	}

	println(a == b)		// true
```

#### 匿名字段

`匿名字段`指没有名字只有类型的字段，可直接使用匿名字段的成员

```go
type person struct {
    name string
    age  int
}

type admin struct {
    person			// 匿名字段，此处只有类型
    level	 int
}

ad := admin{
    person: person{
        name: "aaa",
        age:  10,
    },
    level: 1,
}

println(ad.name)
```

#### 字段签名

`字段标签`是对字段进行描述的元素据，它是类型的组成部分，以``括起来

```go
type user struct {
    name  string  `用户名`
    age   int     `年龄`
}
```

































