# sync.RWMutex - 解决并发读写问题

当多个线程访问共享数据时，会出现并发读写问题（[reader-writer problems](https://en.wikipedia.org/wiki/readers%E2%80%93writers_problem)）。有两种访问数据的线程类型：

- 读线程 reader：只进行数据读取
- 写线程 writer：进行数据修改

当 writer 获取到数据的访问权限后，其他任何线程（reader 或 writer）都无权限访问此数据。这种约束亦存在于现实中，比如，当 writer 在修改数据无法保证原子性时（如数据库），此时读取未完成的修改必须被阻塞，以防止加载脏数据（译者注：数据库中的脏读）。还有许多诸如此类的核心问题，例如：

- writer 不能无限等待
- reader 不能无限等待
- 不允许线程出现无限等待

多读/单写互斥锁（如[sync.RWMutex](https://golang.org/pkg/sync/#RWMutex)）的具体实现解决了一种并发读写问题。接下来，让我们看下在 Go 语言中是如何实现的，同时它提供了哪些的数据可靠性保证机制。

作为额外的工作，我们将深入研究分析竞态情况下的互斥锁。

## 用法

在深入研究实现细节之前，我们先看看`sync.RWMutex`的使用实例。下面的程序使用读写互斥锁来保护临界区--`sleep()`。为了更好的展示整个过程，临界区部分计算了当前正在执行的 reader 和 writer 的数量（[源码](https://play.golang.org/p/xoiqW0RQQE9)）。

```golang
package main

import (
    "fmt"
    "math/rand"
    "strings"
    "sync"
    "time"
)

func init() {
    rand.Seed(time.Now().Unix())
}

func sleep() {
    time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
}

func reader(c chan int, m *sync.RWMutex, wg *sync.WaitGroup) {
    sleep()
    m.RLock()
    c <- 1
    sleep()
    c <- -1
    m.RUnlock()
    wg.Done()
}

func writer(c chan int, m *sync.RWMutex, wg *sync.WaitGroup) {
    sleep()
    m.Lock()
    c <- 1
    sleep()
    c <- -1
    m.Unlock()
    wg.Done()
}

func main() {
    var m sync.RWMutex
    var rs, ws int
    rsCh := make(chan int)
    wsCh := make(chan int)
    go func() {
        for {
            select {
            case n := <-rsCh:
                rs += n
            case n := <-wsCh:
                ws += n
            }
            fmt.Printf("%s%s\n", strings.Repeat("R", rs),
                    strings.Repeat("W", ws))
        }
    }()
    wg := sync.WaitGroup{}
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go reader(rsCh, &m, &wg)
    }
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go writer(wsCh, &m, &wg)
    }
    wg.Wait()
}
```

> play.golang.org 加载的程序环境是确定的（比如开始时间），所以`rand.Seed(time.Now().Unix())`总是返回相同的数值，此时程序的执行结果可能总是相同的。为了避免这种情况，可通过修改不同的随机种子值或者在自己的机器上执行程序。

程序执行结果：

```plain
W

R
RR
RRR
RRRR
RRRRR
RRRR
RRR
RRRR
RRR
RR
R

W

R
RR
RRR
RRRR
RRR
RR
R

W
```

> 译者注：不同机器上运行的结果会有所不同 每次执行完一组 goroutine（reader 和 writer）的临界区代码后，都会打印新的一行。很显然，RWMutex 允许至少一个 reader（一个或多个 reader）存在而 writer 同时只能存在一个。

同样重要且将进一步讨论的是：writer 调用到`Lock()`时，将会使新的 reader/writer 被阻塞。当存在 reader 加了 RLock 时，writer 会等待这一组 reader 完成正在执行的任务，当这一组任务完成后，writer 将开始执行。从输出可以很明显的看到，每一行的 R 都会递减一个，直到没有 R 之后将打印一个 W。

```plain
...
RRRRR
RRRR
RRR
RR
R

W
...
```

一旦 writer 结束，之前被阻塞的 reader 将恢复执行，然后下一个 writer 也将开始启动。值得一提的是，如果一个 writer 完成，并且有 reader 和 writer 都在等待，那么首个 reader 将解除阻塞，然后才轮到 writer。这种交替执行的方式使得 writer 需等待当前这组 reader 完成，所以无论 reader 还是 writer 都不会有无限等待的情况。

## 实现

> 注意，本文针对的`RWMutex`实现([Go commit: 718d6c58](https://github.com/golang/go/blob/718d6c5880fe3507b1d224789b29bc2410fc9da5/src/sync/rwmutex.go))在 Go 不同版本中可能随时有修改。`RWMutex`为 reader 提供两个方法（`RLock`和`RUnlock`）、也为 writer 提供了两个方法（`Lock`和`Unlock`）

## 读锁 RLock

为了简洁起见，我们先跳过源码中竞态检测相关部分（它们将被`...`代替）。

```golang
func (rw *RWMutex) RLock() {
    ...
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {    
        runtime_SemacquireMutex(&rw.ReadeSem, false)
    }
    ...
}
```

`readerCount`字段是`int32`类型的值，表示待处理的 reader 数量（正在读取数据或被 writer 阻塞）。这基本上是已调用 RLock 函数，但尚未调用 RUnlock 函数的 reader 数量。

[atomic.AddInt32](https://golang.org/pkg/sync/atomic/#AddInt32)等价于如下原子性表达：

```golang
*addr += delta
return *addr
```

`addr`是`*int32`类型变量，`delta`是`int32`类型。因为此操作具有原子性，所以累加`delta`操作不会影响其他线程（更多详见[Fetch-and-add](https://en.wikipedia.org/wiki/Fetch-and-add)）。

> 如果没有 writer，则`readerCount`总是会大于或等于 0（译者注：因为 writer 会把 readerCount 置为负数，通过 Lock 函数的 atomic.AddInt32(&rw.readerCount, -rwmutexMaxreaders)，此时 reader 是一种运行速度很快的非阻塞方式，因为只需要调用`atomic.AddInt32`。

## 信号量 Semaphore

信号量是 Edsger Dijkstra 发明的数据结构，在解决多种同步问题时很有用。其本质是一个整数，并关联两个操作：

- 申请`acquire`（也称为 `wait`、`decrement` 或 `P` 操作）
- 释放`release`（也称 `signal`、`increment` 或 `V` 操作）

`acquire`操作将信号量减 1，如果结果值为负则线程阻塞，且直到其他线程进行了信号量累加为正数才能恢复。如结果为正数，线程则继续执行。

`release`操作将信号量加 1，如存在被阻塞的线程，此时他们中的一个线程将解除阻塞。

Go 运行时提供的`runtime_SemacquireMutex`和`runtime_Semrelease`函数可用来实现`sync.RWMutex`互斥锁。

## 锁 Lock

实现源码：

```golang
func (rw *RWMutex) Lock() {
    ...
    rw.w.Lock()
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxreader) + rwmutexMaxreader
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {     
        runtime_SemacquireMutex(&rw.writerSem, false)
    }
    ...
}
```

writer 通过`Lock`方法获取共享数据的独占权限。首先，它会申请阻塞其他写操作的互斥锁（`rw.w.Lock()`），此互斥锁在`Unlock`函数的最后才会进行解锁。下一步，将`readerCount`减去`rwmutexMaxreader`（值为 1 左移 30 位, `1<<30`）使其为负数。当`readerCount`变为负数时，Rlock 将阻塞接下来的所有读请求。

再回过头来看下`Rlock()`函数中逻辑：

```golang
if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    // A writer is pending, wait for it.    
    runtime_SemacquireMutex(&rw.SeadeSem, false)
}
```

后续的 reader 将会被阻塞，那么已运行的 reader 会怎样呢？`readerWait`字段用来记录当前 reader 执行的数量。writer 被信号量`writerSem`阻塞，直到最后一个 reader 在使用后面讨论的`RUnlock`方法解锁后会把`writerSem`加 1，此时信号量将变成 0，`writer`被解除阻塞（译者注：RUnlock 函数中的`runtime_Semrelease(&rw.writerSem, false)`）

如果没有有效的 reader，那么 writer 将继续其执行。

## 最大 reader 数 rwmutexMaxreader

在[rwmutex.go](https://github.com/golang/go/blob/718d6c5880fe3507b1d224789b29bc2410fc9da5/src/sync/rwmutex.go)中定义的常量：

```golang
const rwmutexMaxreader = 1 << 30
```

那么，其用途是什么，以及`1<<30`表示什么意义呢？

`readerCount`字段是[int32](https://golang.org/pkg/builtin/#int32)类型，其范围为：

```plain
[-1 << 31, (1 << 31) — 1] or [-2147483648, 2147483647]
```

`RWMutext`使用此字段来计算挂起的 reader 和 writer 的标记（置为负数）。在`Lock`方法中：

```golang
r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxreader) + rwmutexMaxreader
```

`Lock`会将`readerCount`字段减去`1<<30`，当`readerCount`负值时表示 writer 调用了`Lock`正等待被处理，`atomic.AddInt32(&rw.readerCount, -rwmutexMaxreaders) + rwmutexMaxreaders`这个操作既让`readerCount`变为负数又使`r`存储回了 readerCount。`rwmutexMaxreaders`也可以限制被挂起 reader 的数量。如果有`rwmutexMaxreader`个或更多挂起的 reader，那么`readerCount`将是非负值，此时将导致整个机制的崩溃。所以，reader 实际的限制数是：`rwmutexMaxreader - 1`，此值`1073741823`超过了`10亿`。

## 解读锁 RUnlock

实现源码：

```golang
func (rw *RWMutex) RUnlock() {
    ...
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxreader {
            race.Enable()
            thrSw("sync: RUnlock of unlocked RWMutex")
        }
        // A writer is pending.
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            // The last reader unblocks the writer.       
            runtime_Semrelease(&rw.WriteSem, false)
        }
    }
    ...
}
```

每次调用此方法将使`readerCount`减 1（RLock 方法中增加 1）。如果减完后`readerCount`值为负，则表示当前存在 writer 正在等待或运行。这是因为在`Lock()`方法中`readerCount`减去了`rwmutexMaxreader`。然后，当检查到将完成的 reader 数量（readerWait 数值）最终为 0 时，则表示 writer 可以最终申请信号量。（译者注：`r < 0`时，存在两个分支，当走 r+1 == 0 的分支时，表示 readerCount 此时为 0 即没有 RLock，所以 throw 了。当走下面那个分支时，`r < 0`则是因为存在 writer 把 readerCount 置为了负数在等待 reader 结束，那么当最后一个 reader 解锁时需要将 WriteSem 信号量加 1，唤醒 writer）

## 解锁 Unlock

实现源码：

```golang
func (rw *RWMutex) Unlock() {
    ...
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxreader)
    if r >= rwmutexMaxreader {
        race.Enable()
        throw("sync: Unlock of unlocked RWMutex")
    }
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false)
    }
    rw.w.Unlock()
    ...
}
```

解锁被 writer 持有的互斥锁时，首先通过`atomic.AddInt32`将`readerCount`加上`rwmutexMaxreader`，这时`readerCount`将变成非负值。如`readerCount`比 0 大，则表示存在 reader 正在等待 writer 执行完成，此时应唤醒这些等待的 reader。之后写锁将被释放，从而允许其他 writer 为了写入而锁定互斥锁。（译者注：如果还存在挂起的 reader，则在 writer 解锁之前需要通过信号量 readerSem 唤醒这些 reader 执行）

如果 reader 或 writer 尝试解锁未锁定的互斥锁时，调用`Unlock`或`Runlock`方法将抛出错误（[示例源码](https://play.golang.org/p/YMdFET74olU)）。

```golang
m := sync.RWMutex{}
m.Unlock()
```

输出：

```plain
fatal error: sync: Unlock of unlocked RWMutex
...
```

## 递归读锁定 Recursive read locking

文档描述：

> 如果一个 reader goroutine 持有了读锁，而此时另一个 writer goroutine 调用`Lock`申请加写锁，此后在最初的读锁被释放前其他 goroutine 不能获取到读锁。特定情况下，这能防止递归读锁，这种策略保证了锁的可用性，`Lock`的调用会阻止其他新的 reader 来获得锁。

RWMutex 的工作方式是，如果有一个 writer 调用了 Lock，则所有调用 RLock 都将被锁定，无论是否已经获得了读锁定（[示例源码](https://play.golang.org/p/oHvZh4u3nJl)）: 示例代码：

```golang
package main

import (
    "fmt"
    "sync"
    "time"
)

var m sync.RWMutex

func f(n int) int {
    if n < 1 {
        return 0
    }
    fmt.Println("RLock")
    m.RLock()
    defer func() {
        fmt.Println("RUnlock")
        m.RUnlock()
    }()
    time.Sleep(100 * time.Millisecond)
    return f(n-1) + n
}

func main() {
    done := make(chan int)
    go func() {
        time.Sleep(200 * time.Millisecond)
        fmt.Println("Lock")
        m.Lock()
        fmt.Println("Unlock")
        m.Unlock()
        done <- 1
    }()
    f(4)
    <-done
}
```

输出：

```plain
RLock
RLock
RLock
Lock
RLock
fatal error: all goroutines are asleep - deadlock!
```

译者注（至下一节以前均为译者注）：为什么会发送死锁呢？原作者用递归函数在 defer 里面解锁，那么在加第三层读锁的时候，还没有读锁解锁。这时，readCount 是 3，此时正好加了一个 Lock 写锁，由于 readCount 是 3

```golang
if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
         runtime_Semacquire(&rw.writerSem)
        }
```

由上可知，此时 writer 需要等待所有进行中的 reader 完成，此时又调用了 RLock，

```golang
if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    // A writer is pending, wait for it.
    runtime_Semacquire(&rw.readerSem)
}
```

由于在第四个 RLock 前，加了 Lock 操作，使得 readerCount 为负数。所以就造成了死锁，即 reader 在等待 readerSem，writer 在等待 writerSem*

## 复制锁 Copying locks

`go tool vet`可以检测锁是否被复制了，因为复制锁会导致死锁。更多关于此问题可参考之前的文章：[Detect locks passed by value in Go](https://medium.com/golangspec/detect-locks-passed-by-value-in-go-efb4ac9a3f2b)

## 性能 Performance

之前有人发现，在 CPU 核数增多时 RWMutex 的性能会有下降，详见：[sync: RWMutex scales poorly with CPU count](https://github.com/golang/go/issues/17973)

## 争用 Contention

Go 版本 ≥ 1.8 之后，支持分析争用的互斥锁（[runtime: Profile goroutines holding contended mutexes.](https://github.com/golang/go/commit/ca922b6d363b6ca47822188dcbc5b92d912c7a4b)）。我们来看下如何做：

```golang
package main

import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "sync"
    "time"
)

func main() {
    var mu sync.Mutex
    runtime.SetMutexProfileFraction(5)
    for i := 0; i < 10; i++ {
        go func() {
            for {
                mu.Lock()
                time.Sleep(100 * time.Millisecond)
                mu.Unlock()
            }
        }()
    }
    http.ListenAndServe(":8888", nil)
}
> go build mutexcontention.go
> ./mutexcontention
```

当`mutexcontention`程序运行时，执行 pprof：

```plain
> go tool pprof mutexcontention http://localhost:8888/debug/pprof/mutex?debug=1
Fetching profile over HTTP from http://localhost:8888/debug/pprof/mutex?debug=1
Saved profile in /Users/mlowicki/pprof/pprof.mutexcontention.contentions.delay.003.pb.gz
File: mutexcontention
Type: delay
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) list main
Total: 57.28s
ROUTINE main.main.func1 in .../src/github.com/mlowicki/mutexcontention/mutexcontention.go
0     57.28s (flat, cum)   100% of Total
.          .     14:   for i := 0; i < 10; i++ {
.          .     15:           go func() {
.          .     16:                   for {
.          .     17:                           mu.Lock()
.          .     18:                           time.Sleep(100 * time.Millisecond)
.     57.28s     19:                           mu.Unlock()
.          .     20:                   }
.          .     21:           }()
.          .     22:   }
.          .     23:
.          .     24:   http.ListenAndServe(":8888", nil)
```

注意，为什么这里耗时 57.28s，且指向了`mu.Unlock()`呢？

当 goroutine 调用`Lock`而阻塞时，会记录当前发生的准确时间--叫做`acquiretime`。当另一个 groutine 解锁，至少存在一个 goroutine 在等待获得锁，则其中一个解除阻塞并调用其`mutexevent`函数。该`mutexevent`函数通过检查`SetMutexProfileFraction`设置的速率来决定此事件应被保留还是丢弃。此事件包含整个等待的时间（当前时间 - 获得时间）。从上面的例子可以看出，所有阻塞在特定互斥锁的 goroutines 的总等待时间会被收集和展示。

在 Go 1.11（[sync: enable profiling of RWMutex](https://github.com/golang/go/commit/88ba64582703cea0d66a098730215554537572de)）中将增加读锁（Rlock 和 RUnlock）的争用。