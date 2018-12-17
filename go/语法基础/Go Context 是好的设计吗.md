# Golang Context 是好的设计吗？

## 1、What Context

Context是Golang官方定义的一个package，它定义了Context类型，里面包含了Deadline/Done/Err方法以及绑定到Context上的成员变量值Value，具体定义如下：

```
type Context interface {
    // 返回Context的超时时间（超时返回场景）
    Deadline() (deadline time.Time, ok bool)
    // 在Context超时或取消时（即结束了）返回一个关闭的channel
    // 即如果当前Context超时或取消时，Done方法会返回一个channel，然后其他地方就可以通过判断Done方法是否有返回（channel），如果有则说明Context已结束
    // 故其可以作为广播通知其他相关方本Context已结束，请做相关处理。
    Done() <-chan struct{}

    // 返回Context取消的原因
    Err() error
    
    // 返回Context相关数据
    Value(key interface{}) interface{}
}
```

那么到底什么Context？

可以字面意思可以理解为上下文，比较熟悉的有进程/线程上线文，关于Golang中的上下文，一句话概括就是：goroutine的相关环境快照，其中包含函数调用以及涉及的相关的变量值。
通过Context可以区分不同的goroutine请求，因为在Golang Severs中，每个请求都是在单个goroutine中完成的。

**注**：关于goroutine的理解可以移步[这里](https://www.zhihu.com/question/20862617)。

## 2、Why Context

由于在Golang severs中，每个request都是在单个goroutine中完成，并且在单个goroutine（不妨称之为A）中也会有请求其他服务（启动另一个goroutine（称之为B）去完成）的场景，这就会涉及多个Goroutine之间的调用。如果某一时刻请求其他服务被取消或者超时，则作为深陷其中的当前goroutine B需要立即退出，然后系统才可回收B所占用的资源。
即一个request中通常包含多个goroutine，这些goroutine之间通常会有交互。
![图片描述](img/501250143-5c1654d795ca7_articlex.png)

那么，如何有效管理这些goroutine成为一个问题（主要是退出通知和元数据传递问题），Google的解决方法是Context机制，相互调用的goroutine之间通过传递context变量保持关联，这样在不用暴露各goroutine内部实现细节的前提下，有效地控制各goroutine的运行。
![图片描述](img/4049495135-5c1654e633d21_articlex.png)

如此一来，通过传递Context就可以追踪goroutine调用树，并在这些调用树之间传递通知和元数据。
虽然goroutine之间是平行的，没有继承关系，但是Context设计成是包含父子关系的，这样可以更好的描述goroutine调用之间的树型关系。

## 3、How to use

生成一个Context主要有两类方法：

### 3.1 ）顶层Context：Background

要创建Context树，首先就是要创建根节点

```
// 返回一个空的Context，它作为所有由此继承Context的根节点
func Background() Context
```

该Context通常由接收request的第一个goroutine创建，它不能被取消、没有值、也没有过期时间，常作为处理request的顶层context存在。

### 3.2）下层Context：WithCancel/WithDeadline/WithTimeout

有了根节点之后，接下来就是创建子孙节点。为了可以很好的控制子孙节点，Context包提供的创建方法均是带有第二返回值（CancelFunc类型），它相当于一个Hook，在子goroutine执行过程中，可以通过触发Hook来达到控制子goroutine的目的（通常是取消，即让其停下来）。再配合Context提供的Done方法，子goroutine可以检查自身是否被父级节点Cancel：

```
select { 
    case <-ctx.Done(): 
        // do some clean… 
}
```

**注**：父节点Context可以主动通过调用cancel方法取消子节点Context，而子节点Context只能被动等待。同时父节点Context自身一旦被取消（如其上级节点Cancel），其下的所有子节点Context均会自动被取消。

有三种创建方法：

```
// 带cancel返回值的Context，一旦cancel被调用，即取消该创建的context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) 

// 带有效期cancel返回值的Context，即必须到达指定时间点调用的cacel方法才会被执行
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) 

// 带超时时间cancel返回值的Context，类似Deadline，前者是时间点，后者为时间间隔
// 相当于WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

下面来看改编自Advanced Go Concurrency Patterns视频提供的一个简单例子：

```
package main

import (
    "context"
    "fmt"
    "time"
)

func someHandler() {
    // 创建继承Background的子节点Context
    ctx, cancel := context.WithCancel(context.Background())
    go doSth(ctx)

    //模拟程序运行 - Sleep 5秒
    time.Sleep(5 * time.Second)
    cancel()
}

//每1秒work一下，同时会判断ctx是否被取消，如果是就退出
func doSth(ctx context.Context) {
    var i = 1
    for {
        time.Sleep(1 * time.Second)
        select {
        case <-ctx.Done():
            fmt.Println("done")
            return
        default:
            fmt.Printf("work %d seconds: \n", i)
        }
        i++
    }
}

func main() {
    fmt.Println("start...")
    someHandler()
    fmt.Println("end.")
}
```

输出结果：

![clipboard.png](img/1115608585-5c16566625202_articlex.png)

注意，此时doSth方法中case之done的`fmt.Println("done")`并没有被打印出来。

超时场景：

```
package main

import (
    "context"
    "fmt"
    "time"
)

func timeoutHandler() {
    // 创建继承Background的子节点Context
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    go doSth(ctx)

    //模拟程序运行 - Sleep 10秒
    time.Sleep(10 * time.Second)
    cancel() // 3秒后将提前取消 doSth goroutine
}

//每1秒work一下，同时会判断ctx是否被取消，如果是就退出
func doSth(ctx context.Context) {
    var i = 1
    for {
        time.Sleep(1 * time.Second)
        select {
        case <-ctx.Done():
            fmt.Println("done")
            return
        default:
            fmt.Printf("work %d seconds: \n", i)
        }
        i++
    }
}

func main() {
    fmt.Println("start...")
    timeoutHandler()
    fmt.Println("end.")
}
```

输出结果：

![clipboard.png](img/4037997361-5c1657a9eed9a_articlex.png)

## 4、Really elegant solution?

前面铺地了这么多。

确实，通过引入Context包，一个request范围内所有goroutine运行时的取消可以得到有效的控制。但是这种解决方式却不够优雅。

### 4.1 Like a virus

一旦代码中某处用到了Context，传递Context变量（通常作为函数的第一个参数）会像病毒一样蔓延在各处调用它的地方。比如在一个request中实现数据库事务或者分布式日志记录，创建的context，会作为参数传递到任何有数据库操作或日志记录需求的函数代码处。即每一个相关函数都必须增加一个context.Context类型的参数，且作为第一个参数，这对无关代码完全是侵入式的。

更多详细内容可参见：Michal Strba 的[context-should-go-away-go2](https://faiface.github.io/post/context-should-go-away-go2/)文章

Google Group上的讨论可移步[这里](https://groups.google.com/forum/#!searchin/golang-nuts/transaction%7Csort:date/golang-nuts/eEDlXAVW9vU/IChp34xpCQAJ)。

### 4.2 Context isn’t for cancellation

Context机制最核心的功能是在goroutine之间传递cancel信号，但是它的实现是不完全的。

Cancel可以细分为主动与被动两种，通过传递context参数，让调用goroutine可以主动cancel被调用goroutine。但是如何得知被调用goroutine什么时候执行完毕，这部分Context机制是没有实现的。而现实中的确又有一些这样的场景，比如一个组装数据的goroutine必须等待其他goroutine完成才可开始执行，这是context明显不够用了，必须借助sync.WaitGroup。

```
func serve(l net.Listener) error {
        var wg sync.WaitGroup
        var conn net.Conn
        var err error
        for {
                conn, err = l.Accept()
                if err != nil {
                        break
                }
                wg.Add(1)
                go func(c net.Conn) {
                        defer wg.Done()
                        handle(c)
                }(conn)
        }
        wg.Wait()
        return err
}
```

### 4.3 context.value

context.Value相当于goroutine的TLS（Thread Local Storage），但它不是静态类型安全的，任何结构体变量都必须作为字符串形式存储。同时，所有context都会在其中定义变量，很容易造成命名冲突。