# Golang 反射

## 基本了解

在Go语言中，大多数时候值/类型/函数非常直接，要的话，定义一个。你想要个`Struct`

```
type Foo struct {
    A int 
    B string
}
```

你想要一个值，你定义出来

```
var x Foo
```

你想要一个函数，你定义出来

```
func DoSomething(f Foo) {
  fmt.Println(f.A, f.B)
}
```

但是有些时候，你需要搞一些运行时才能确定的东西，比如你要从文件或者网络中获取一些字典数据。又或者你要搞一些不同类型的数据。在这种情况下，`reflection`就有用啦。**reflection能够让你拥有以下能力**

- 在运行时检查type
- 在运行时检查/修改/创建 值/函数/结构
   总的来说，go的`reflection`围绕者三个概念`Types`, `Kinds`, `Values`。 所有关于反射的操作都在`reflect`包里面

## 反射的Power

### Type的Power

首先，我们看看如何通过反射来获取值得类型。

```
varType := reflect.TypeOf(var)
```

从反射接口可以看到有一大堆得函数等着我们去用。可以从注释里面看到。反射包默认我们知道我们要干啥子，比如`varType.Elem()`就会`panic`。因为Elem()只有`Array, Chan, Map, Ptr, or Slice.`这些类型才有这个方法。具体可以查看测试代码。通过运行以下代码可查看所有reflect函数的示例

```
package main

import (
    "fmt"
    "reflect"
)

type FooIF interface {
    DoSomething()
    DoSomethingWithArg(a string)
    DoSomethingWithUnCertenArg(a ... string)
}

type Foo struct {
    A int
    B string
    C struct {
        C1 int
    }
}

func (f *Foo) DoSomething() {
    fmt.Println(f.A, f.B)
}

func (f *Foo) DoSomethingWithArg(a string) {
    fmt.Println(f.A, f.B, a)
}

func (f *Foo) DoSomethingWithUnCertenArg(a ... string) {
    fmt.Println(f.A, f.B, a[0])
}

func (f *Foo) returnOneResult() int {
    return 2
}

func main() {
    var simpleObj Foo
    var pointer2obj = &simpleObj
    var simpleIntArray = [3]int{1, 2, 3}
    var simpleMap = map[string]string{
        "a": "b",
    }
    var simpleChan = make(chan int, 1)
    var x uint64
    var y uint32

    varType := reflect.TypeOf(simpleObj)
    varPointerType := reflect.TypeOf(pointer2obj)

    // 对齐之后要多少容量
    fmt.Println("Align: ", varType.Align())
    // 作为结构体的`field`要对其之后要多少容量
    fmt.Println("FieldAlign: ", varType.FieldAlign())
    // 叫啥
    fmt.Println("Name: ", varType.Name())
    // 绝对引入路径
    fmt.Println("PkgPath: ", varType.PkgPath())
    // 实际上用了多少内存
    fmt.Println("Size: ", varType.Size())
    // 到底啥类型的
    fmt.Println("Kind: ", varType.Kind())

    // 有多少函数
    fmt.Println("NumMethod: ", varPointerType.NumMethod())

    // 通过名字获取一个函数
    m, success := varPointerType.MethodByName("DoSomethingWithArg")
    if success {
        m.Func.Call([]reflect.Value{
            reflect.ValueOf(pointer2obj),
            reflect.ValueOf("sad"),
        })
    }

    // 通过索引获取函数
    m = varPointerType.Method(1)
    m.Func.Call([]reflect.Value{
        reflect.ValueOf(pointer2obj),
        reflect.ValueOf("sad2"),
    })

    // 是否实现了某个接口
    fmt.Println("Implements:", varPointerType.Implements(reflect.TypeOf((*FooIF)(nil)).Elem()))

    //  看看指针多少bit
    fmt.Println("Bits: ", reflect.TypeOf(x).Bits())

    // 查看array, chan, map, ptr, slice的元素类型
    fmt.Println("Elem: ", reflect.TypeOf(simpleIntArray).Elem().Kind())

    // 查看Array长度
    fmt.Println("Len: ", reflect.TypeOf(simpleIntArray).Len())

    // 查看结构体field
    fmt.Println("Field", varType.Field(1))

    // 查看结构体field
    fmt.Println("FieldByIndex", varType.FieldByIndex([]int{2, 0}))

    // 查看结构提field
    fi, success2 := varType.FieldByName("A")
    if success2 {
        fmt.Println("FieldByName", fi)
    }

    // 查看结构体field
    fi, success2 = varType.FieldByNameFunc(func(fieldName string) bool {
        return fieldName == "A"
    })
    if success2 {
        fmt.Println("FieldByName", fi)
    }

    //  查看结构体数量
    fmt.Println("NumField", varType.NumField())

    // 查看map的key类型
    fmt.Println("Key: ", reflect.TypeOf(simpleMap).Key().Name())

    // 查看函数有多少个参数
    fmt.Println("NumIn: ", reflect.TypeOf(pointer2obj.DoSomethingWithUnCertenArg).NumIn())

    // 查看函数参数的类型
    fmt.Println("In: ", reflect.TypeOf(pointer2obj.DoSomethingWithUnCertenArg).In(0))

    // 查看最后一个参数，是否解构了
    fmt.Println("IsVariadic: ", reflect.TypeOf(pointer2obj.DoSomethingWithUnCertenArg).IsVariadic())

    // 查看函数有多少输出
    fmt.Println("NumOut: ", reflect.TypeOf(pointer2obj.DoSomethingWithUnCertenArg).NumOut())

    // 查看函数输出的类型
    fmt.Println("Out: ", reflect.TypeOf(pointer2obj.returnOneResult).Out(0))

    // 查看通道的方向, 3双向。
    fmt.Println("ChanDir: ", int(reflect.TypeOf(simpleChan).ChanDir()))

    // 查看该类型是否可以比较。不能比较的slice, map, func
    fmt.Println("Comparable: ", varPointerType.Comparable())

    // 查看类型是否可以转化成另外一种类型
    fmt.Println("ConvertibleTo: ", varPointerType.ConvertibleTo(reflect.TypeOf("a")))

    // 该类型的值是否可以另外一个类型
    fmt.Println("AssignableTo: ", reflect.TypeOf(x).AssignableTo(reflect.TypeOf(y)))
}
```

### Value的Power

除了检查变量的类型，你可以通过`reflection`来读/写/新建一个值。不过首先先获取反射值类型

```
refVal := reflect.ValueOf(var) 
```

如果你想要修改变量的值。你需要获取反射指向该变量的指针，具体原因后面解释

```
refPtrVal := reflect.ValueOf(&var)
```

当然你有了`reflect.Value`,通过`Type()`方法可以很容易的获取`reflect.Type`。如果要改变该变量的值用

```
refPtrVal.Elem().Set(newRefValue)
```

当然`Set`方法的参数必须也得是`reflect.Value`
 如果你想创建一个新的值，用以下下代码

```
newPtrVal := reflect.New(varType)
```

然后在用`Elem().Set()`来进行值的初始化。当然还有不同的value有一大堆的不同的方法。这里就不写了。我们重点看看下面这段官方例子

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // swap is the implementation passed to MakeFunc.
    // It must work in terms of reflect.Values so that it is possible
    // to write code without knowing beforehand what the types
    // will be.
    swap := func(in []reflect.Value) []reflect.Value {
        return []reflect.Value{in[1], in[0]}
    }

    // makeSwap expects fptr to be a pointer to a nil function.
    // It sets that pointer to a new function created with MakeFunc.
    // When the function is invoked, reflect turns the arguments
    // into Values, calls swap, and then turns swap's result slice
    // into the values returned by the new function.
    makeSwap := func(fptr interface{}) {
        // fptr is a pointer to a function.
        // Obtain the function value itself (likely nil) as a reflect.Value
        // so that we can query its type and then set the value.
        fn := reflect.ValueOf(fptr).Elem()

        // Make a function of the right type.
        v := reflect.MakeFunc(fn.Type(), swap)

        // Assign it to the value fn represents.
        fn.Set(v)
    }

    // Make and call a swap function for ints.
    var intSwap func(int, int) (int, int)
    makeSwap(&intSwap)
    fmt.Println(intSwap(0, 1))

    // Make and call a swap function for float64s.
    var floatSwap func(float64, float64) (float64, float64)
    makeSwap(&floatSwap)
    fmt.Println(floatSwap(2.72, 3.14))

}
```

## 原理

### 认清楚type与interface

go是一个静态类型语言，每一个变量有`static type`，比如`int`，`float`，何谓`static type`，我的理解是一定长度的二进制块与解释。比如同样的二进制块`00000001` 在bool类型中意思是`true`。而在int类型中解释是`1`.   我们看看以下这个最简单的例子

```
type MyInt int
var i int
var j MyInt
```

i,j在内存中都是用int这一个底层类型来表示，但是在实际编码过程中，在编译的时候他们并非一个类型，你不能直接将i的值赋给j。是不是有点奇怪，你执行的时候编译器会告诉你，你不能将MyInt类型的值赋给int类型的值。这个`type`不是`class`也不是`python`的`type`.

`interface`作为一种特殊的`type`, 表示方法的集合。一个`interface`的值可以存任何确定的值只要这个值实现了`interface`的方法。`interface{}`某些时候和Java的`Object`好想，实际上`interface`是有两部分内容组成的，**实际的值**和**值的具体类型**。这也可以解释为什么下面这段代码和其他语言都不一样。具体关于interface的原理可以参考[go data structures: interfaces](https://research.swtch.com/2009/12/go-data-structures-interfaces.html)。

```
package main

import (
    "fmt"
)

type A interface {
    x(param int)
}

type B interface {
    y(param int)
}


type AB struct {

}

func (ab *AB) x(param int) {
    fmt.Printf("%p", ab)
    fmt.Println(param)
}

func (ab *AB) y(param int) {
    fmt.Printf("%p", ab)
    fmt.Println(param)
}


func printX(a A){
    fmt.Printf("%p", a)
    a.x(2)
}

func printY(b B){
    fmt.Printf("%p", b)
    b.y(3)
}



func main() {
    var ab = new(AB)
    printX(ab)
    printY(ab)


    var aInfImpl A
    var bInfImpl B

    aInfImpl = new(AB)
    //bInfImpl = aInfImpl  会报错
    bInfImpl = aInfImpl.(B)
    bInfImpl.y(2)
}
```

### golang反射三定理

#### 把一个`interface`值，拆分出反射对象

反射仅仅用于检查接口值的(Value, Type)。如上一章提到的两个方法`ValueOf`和`TypeOf`。通过`ValueOf`我门可以轻易的拿到`Type`

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

这段代码输出

```
type: float64
```

那么问题就来了，接口在哪里？只是申明了一个`float64`的变量。哪里来的`interface`。有的，答案就藏在 `TypeOf`参数里面

```
func TypeOf(i interface{}) Type
```

当我们调用`reflect.TypeOf(x)`, x首先被存在一个空的`interface`里面。然后在被当作参数传到函数执行栈内。** `reflect.TypeOf`解开这个`interface`的`pair`然后恢复出类型信息**

#### 把反射对象组合成一个接口值

就像镜面反射一样，go的反射是可逆的。给我一个`reflect.Value`。我们能够恢复出一个`interface`的值。事实上，以下函数干的事情就是将`Value`和`Type`组狠起来塞到 `interface`里面去。所以我们可以

```
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

接下来就是见证奇迹的时刻。`fmt.Println`和`fmt.Printf`的参数都是`interface{}`。我们真正都不需要将上面例子的`y`转化成明确的`float64`。我就可以去打印他比如

```
fmt.Println(v.Interface())
```

甚至我们的`interface`藏着的那个`type`是`float64`。我们可以直接用占位符来打印

```
fmt.Println("Value is %7.le\n", v.Interface())
```

再重复一边，没有必要将`v.Interface()`的类型强转到`float64`;这个空的`interface{}`包含了`concrete type`。函数调用会恢复出来

### 要改变一个反射对象，其值必须是可设置的

第三条比较让你比较困惑。不过如果我们理解了第一条，那么这条其实非常好理解。先看一下下面这个例子

```
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

如果执行这段代码，你会发现出现`panic`以下信息

```
panic: reflect.Value.SetFloat using unaddressable value
```

可设置性是一个好东西，但不是所有`reflect.Value`都有他...可以通过`CanSet` 函数来获取是否可设置

```
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

那么到底为什么要有一个可设置性呢？可寻址才可设置，我们在用`reflect.ValueOf`时候，实际上是函数传值。获取x的反射对象，实际上是另外一个`float64`的内存的反射对象。这个时候我们再去设置该反射对象的值，没有意义。这段内存并不是你申明的那个x。