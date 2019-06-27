## 源码剖析 golang 中 sync.Mutex

go 语言以并发作为其特性之一，并发必然会带来对于资源的竞争，这时候我们就需要使用 go 提供的 `sync.Mutex` 这把互斥锁来保证临界资源的访问互斥。

既然经常会用这把锁，那么了解一下其内部实现，就能了解这把锁适用什么场景，特性如何了。



## 引子

在我第一次看这段代码的时候，感觉真的是惊为天人，特别是整个 `Mutex` 只用到了两个私有字段，以及一次 CAS 就加锁的过程，这其中设计以及编程的理念真的让我感觉自愧不如。

在看 `sync.Mutex` 的代码的时候，一定要记住，同时会有多个 goroutine 会来要这把锁，所以锁的状态 `state` 是可能会一直更改的。

## 锁的性质

先说结论：`sync.Mutex` 是把公平锁。

在源代码中，有一段注释：

```
// Mutex fairness.
//
// Mutex can be in 2 modes of operations: normal and starvation.
// In normal mode waiters are queued in FIFO order, but a woken up waiter
// does not own the mutex and competes with new arriving goroutines over
// the ownership. New arriving goroutines have an advantage -- they are
// already running on CPU and there can be lots of them, so a woken up
// waiter has good chances of losing. In such case it is queued at front
// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
// it switches mutex to the starvation mode.
//
// In starvation mode ownership of the mutex is directly handed off from
// the unlocking goroutine to the waiter at the front of the queue.
// New arriving goroutines don't try to acquire the mutex even if it appears
// to be unlocked, and don't try to spin. Instead they queue themselves at
// the tail of the wait queue.
//
// If a waiter receives ownership of the mutex and sees that either
// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
// it switches mutex back to normal operation mode.
//
// Normal mode has considerably better performance as a goroutine can acquire
// a mutex several times in a row even if there are blocked waiters.
// Starvation mode is important to prevent pathological cases of tail latency.
```

看懂这段注释对于我们理解 mutex 这把锁有很大的帮助，这里面讲了这把锁的设计理念。大致意思如下：

```
// 公平锁
//
// 锁有两种模式：正常模式和饥饿模式。
// 在正常模式下，所有的等待锁的goroutine都会存在一个先进先出的队列中（轮流被唤醒）
// 但是一个被唤醒的goroutine并不是直接获得锁，而是仍然需要和那些新请求锁的（new arrivial）
// 的goroutine竞争，而这其实是不公平的，因为新请求锁的goroutine有一个优势——它们正在CPU上
// 运行，并且数量可能会很多。所以一个被唤醒的goroutine拿到锁的概率是很小的。在这种情况下，
// 这个被唤醒的goroutine会加入到队列的头部。如果一个等待的goroutine有超过1ms（写死在代码中）
// 都没获取到锁，那么就会把锁转变为饥饿模式。
//
// 在饥饿模式中，锁的所有权会直接从释放锁(unlock)的goroutine转交给队列头的goroutine，
// 新请求锁的goroutine就算锁是空闲状态也不会去获取锁，并且也不会尝试自旋。它们只是排到队列的尾部。
//
// 如果一个goroutine获取到了锁之后，它会判断以下两种情况：
// 1. 它是队列中最后一个goroutine；
// 2. 它拿到锁所花的时间小于1ms；
// 以上只要有一个成立，它就会把锁转变回正常模式。

// 正常模式会有比较好的性能，因为即使有很多阻塞的等待锁的goroutine，
// 一个goroutine也可以尝试请求多次锁。
// 饥饿模式对于防止尾部延迟来说非常的重要。
```

在下一步真正看源代码之前，我们必须要理解一点：当一个 goroutine 获取到锁的时候，有可能没有竞争者，也有可能会有很多竞争者，那么我们就需要站在不同的 goroutine 的角度上去考虑 goroutine 看到的锁的状态和实际状态、期望状态之间的转化。

## 字段定义

`sync.Mutex` 只包含两个字段：

```
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema	uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	starvationThresholdNs = 1e6
)
```

其中 `state` 是一个表示锁的状态的字段，这个字段会同时被多个 goroutine 所共用（使用 atomic.CAS 来保证原子性），第 0 个 bit（1）表示锁已被获取，也就是已加锁，被某个 goroutine 拥有；第 1 个 bit（2）表示有 goroutine 被唤醒，尝试获取锁；第 2 个 bit（4）标记这把锁是否为饥饿状态。

`sema` 字段就是用来唤醒 goroutine 所用的信号量。

## Lock

在看代码之前，我们需要有一个概念：每个 goroutine 也有自己的状态，存在局部变量里面（也就是函数栈里面），goroutine 有可能是新到的、被唤醒的、正常的、饥饿的。

### atomic.CAS

先瞻仰一下惊为天人的一行代码加锁的 CAS 操作：

```
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	...
}
```

这是第一段代码，这段代码调用了 `atomic` 包中的 `CompareAndSwapInt32` 这个方法来尝试快速获取锁，这个方法的签名如下：

```
// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

意思是，如果 addr 指向的地址中存的值和 old 一样，那么就把 addr 中的值改为 new 并返回 true；否则什么都不做，返回 false。由于是 `atomic` 中的函数，所以是保证了原子性的。

我们来具体看看 CAS 的实现（`src/runtime/internal/atomic/asm_amd64.s`）：

```
// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	} else
//		return 0;
// 这里参数及返回值大小加起来是17，是因为一个指针在amd64下是8字节，
// 然后int32分别是占用4字节，最后的返回值是bool占用1字节，所以加起来是17
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17 
	// 为什么不把*val指针放到AX中呢？因为AX有特殊用处，
	// 在下面的CMPXCHGL里面，会从AX中读取要比较的其中一个数
	MOVQ	ptr+0(FP), BX
	// 所以AX要用来存参数old
	MOVL	old+8(FP), AX
	// 把new中的数存到寄存器CX中
	MOVL	new+12(FP), CX
	// 注意这里了，这里使用了LOCK前缀，所以保证操作是原子的
	LOCK
	// 0(BX) 可以理解为 *val
	// 把 AX中的数 和 第二个操作数 0(BX)——也就是BX寄存器所指向的地址中存的值 进行比较
	// 如果相等，就把 第一个操作数 CX寄存器中存的值 赋给 第二个操作数 BX寄存器所指向的地址
	// 并将标志寄存器ZF设为1
	// 否则将标志寄存器ZF清零
	CMPXCHGL	CX, 0(BX)
	// SETE的作用是：
	// 如果Zero Flag标志寄存器为1，那么就把操作数设为1
	// 否则把操作数设为0
	// 也就是说，如果上面的比较相等了，就返回true，否则为false
	// ret+16(FP)代表了返回值的地址
	SETEQ	ret+16(FP)
	RET
```

如果看不懂也没太大关系，只要知道这个函数的作用，以及这个函数是原子性的即可。

那么这段代码的意思就是：先看看这把锁是不是空闲状态，如果是的话，直接原子性地修改一下 `state` 为已被获取就行了。多么简洁（虽然后面的代码并不是……）！

### 主流程

接下来具体看主流程的代码，代码中有一些位运算看起来比较晕，我会试着用伪代码在边上注释。

```
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}

	// 用来存当前goroutine等待的时间
	var waitStartTime int64
	// 用来存当前goroutine是否饥饿
	starving := false
	// 用来存当前goroutine是否已唤醒
	awoke := false
	// 用来存当前goroutine的循环次数(想一想一个goroutine如果循环了2147483648次咋办……)
	iter := 0
	// 复制一下当前锁的状态
	old := m.state
	// 自旋
	for {
		// 如果是饥饿情况之下，就不要自旋了，因为锁会直接交给队列头部的goroutine
		// 如果锁是被获取状态，并且满足自旋条件（canSpin见后文分析），那么就自旋等锁
		// 伪代码：if isLocked() and isNotStarving() and canSpin()
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 将自己的状态以及锁的状态设置为唤醒，这样当Unlock的时候就不会去唤醒其它被阻塞的goroutine了
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			// 进行自旋(分析见后文)
			runtime_doSpin()
			iter++
			// 更新锁的状态(有可能在自旋的这段时间之内锁的状态已经被其它goroutine改变)
			old = m.state
			continue
		}
		
		// 当走到这一步的时候，可能会有以下的情况：
		// 1. 锁被获取+饥饿
		// 2. 锁被获取+正常
		// 3. 锁空闲+饥饿
		// 4. 锁空闲+正常
		
		// goroutine的状态可能是唤醒以及非唤醒
		
		// 复制一份当前的状态，目的是根据当前状态设置出期望的状态，存在new里面，
		// 并且通过CAS来比较以及更新锁的状态
		// old用来存锁的当前状态
		new := old

		// 如果说锁不是饥饿状态，就把期望状态设置为被获取(获取锁)
		// 也就是说，如果是饥饿状态，就不要把期望状态设置为被获取
		// 新到的goroutine乖乖排队去
		// 伪代码：if isNotStarving()
		if old&mutexStarving == 0 {
			// 伪代码：newState = locked
			new |= mutexLocked
		}
		// 如果锁是被获取状态，或者饥饿状态
		// 就把期望状态中的等待队列的等待者数量+1(实际上是new + 8)
		// (会不会可能有三亿个goroutine等待拿锁……)
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// 如果说当前的goroutine是饥饿状态，并且锁被其它goroutine获取
		// 那么将期望的锁的状态设置为饥饿状态
		// 如果锁是释放状态，那么就不用切换了
		// Unlock期望一个饥饿的锁会有一些等待拿锁的goroutine，而不只是一个
		// 这种情况下不会成立
		if starving && old&mutexLocked != 0 {
			// 期望状态设置为饥饿状态
			new |= mutexStarving
		}
		// 如果说当前goroutine是被唤醒状态，我们需要reset这个状态
		// 因为goroutine要么是拿到锁了，要么是进入sleep了
		if awoke {
			// 如果说期望状态不是woken状态，那么肯定出问题了
			// 这里看不懂没关系，wake的逻辑在下面
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// 这句就是把new设置为非唤醒状态
			// &^的意思是and not
			new &^= mutexWoken
		}
		// 通过CAS来尝试设置锁的状态
		// 这里可能是设置锁，也有可能是只设置为饥饿状态和等待数量
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			// 如果说old状态不是饥饿状态也不是被获取状态
			// 那么代表当前goroutine已经通过CAS成功获取了锁
			// (能进入这个代码块表示状态已改变，也就是说状态是从空闲到被获取)
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// 如果之前已经等待过了，那么就要放到队列头
			queueLifo := waitStartTime != 0
			// 如果说之前没有等待过，就初始化设置现在的等待时间
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			// 既然获取锁失败了，就使用sleep原语来阻塞当前goroutine
			// 通过信号量来排队获取锁
			// 如果是新来的goroutine，就放到队列尾部
			// 如果是被唤醒的等待锁的goroutine，就放到队列头部
			runtime_SemacquireMutex(&m.sema, queueLifo)
			
			// 这里sleep完了，被唤醒
			
			// 如果当前goroutine已经是饥饿状态了
			// 或者当前goroutine已经等待了1ms（在上面定义常量）以上
			// 就把当前goroutine的状态设置为饥饿
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			// 再次获取一下锁现在的状态
			old = m.state
			// 如果说锁现在是饥饿状态，就代表现在锁是被释放的状态，当前goroutine是被信号量所唤醒的
			// 也就是说，锁被直接交给了当前goroutine
			if old&mutexStarving != 0 {
				// 如果说当前锁的状态是被唤醒状态或者被获取状态，或者说等待的队列为空
				// 那么是不可能的，肯定是出问题了，因为当前状态肯定应该有等待的队列，锁也一定是被释放状态且未唤醒
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				// 当前的goroutine获得了锁，那么就把等待队列-1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 如果当前goroutine非饥饿状态，或者说当前goroutine是队列中最后一个goroutine
				// 那么就退出饥饿模式，把状态设置为正常
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				// 原子性地加上改动的状态
				atomic.AddInt32(&m.state, delta)
				break
			}
			// 如果锁不是饥饿模式，就把当前的goroutine设为被唤醒
			// 并且重置iter(重置spin)
			awoke = true
			iter = 0
		} else {
			// 如果CAS不成功，也就是说没能成功获得锁，锁被别的goroutine获得了或者锁一直没被释放
			// 那么就更新状态，重新开始循环尝试拿锁
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

以上为什么 CAS 能拿到锁呢？因为 CAS 会原子性地判断 `old state` 和当前锁的状态是否一致；而总有一个 goroutine 会满足以上条件成功拿锁。

### canSpin

接下来我们来看看上文提到的 `canSpin` 条件如何：

```
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// 这里的active_spin是个常量，值为4
	// 简单来说，sync.Mutex是有可能被多个goroutine竞争的，所以不应该大量自旋(消耗CPU)
	// 自旋的条件如下：
	// 1. 自旋次数小于active_spin(这里是4)次；
	// 2. 在多核机器上；
	// 3. GOMAXPROCS > 1并且至少有一个其它的处于运行状态的P；
	// 4. 当前P没有其它等待运行的G；
	// 满足以上四个条件才可以进行自旋。
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

所以可以看出来，并不是一直无限自旋下去的，当自旋次数到达 4 次或者其它条件不符合的时候，就改为信号量拿锁了。

### doSpin

然后我们来看看 `doSpin` 的实现（其实也没啥好看的）：

```
//go:linkname sync_runtime_doSpin sync.runtime_doSpin
//go:nosplit
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}
```

这是一个汇编实现的函数，简单看两眼 amd64 上的实现：

```
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

看起来没啥好看的，直接跳过吧。

## Unlock

接下来我们来看看 Unlock 的实现，对于 Unlock 来说，有两个比较关键的特性：

1. 如果说锁不是处于 locked 状态，那么对锁执行 Unlock 会导致 panic；
2. 锁和 goroutine 没有对应关系，所以我们完全可以在 goroutine 1 中获取到锁，然后在 goroutine 2 中调用 Unlock 来释放锁（这是什么骚操作！）（虽然不推荐大家这么干……）

```
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	// 这里获取到锁的状态，然后将状态减去被获取的状态(也就是解锁)，称为new(期望)状态
	// 注意以上两个操作是原子的，所以不用担心多个goroutine并发的问题
	new := atomic.AddInt32(&m.state, -mutexLocked)
	// 如果说，期望状态加上被获取的状态，不是被获取的话
	// 那么就panic
	// 在这里给大家提一个问题：干嘛要这么大费周章先减去再加上，直接比较一下原来锁的状态是否被获取不就完事了？
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	// 如果说new状态(也就是锁的状态)不是饥饿状态
	if new&mutexStarving == 0 {
		// 复制一下原先状态
		old := new
		for {
			// 如果说锁没有等待拿锁的goroutine
			// 或者锁被获取了(在循环的过程中被其它goroutine获取了)
			// 或者锁是被唤醒状态(表示有goroutine被唤醒，不需要再去尝试唤醒其它goroutine)
			// 或者锁是饥饿模式(会直接转交给队列头的goroutine)
			// 那么就直接返回，啥都不用做了
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// 走到这一步的时候，说明锁目前还是空闲状态，并且没有goroutine被唤醒且队列中有goroutine等待拿锁
			// 那么我们就要把锁的状态设置为被唤醒，等待队列-1
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			// 又是熟悉的CAS
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				// 如果状态设置成功了，我们就通过信号量去唤醒goroutine
				runtime_Semrelease(&m.sema, false)
				return
			}
			// 循环结束的时候，更新一下状态，因为有可能在执行的过程中，状态被修改了(比如被Lock改为了饥饿状态)
			old = m.state
		}
	} else {
		// 如果是饥饿状态下，那么我们就直接把锁的所有权通过信号量移交给队列头的goroutine就好了
		// handoff = true表示直接把锁交给队列头部的goroutine
		// 注意：在这个时候，锁被获取的状态没有被设置，会由被唤醒的goroutine在唤醒后设置
		// 但是当锁处于饥饿状态的时候，我们也认为锁是被获取的(因为我们手动指定了获取的goroutine)
		// 所以说新来的goroutine不会尝试去获取锁(在Lock中有体现)
		runtime_Semrelease(&m.sema, true)
	}
}
```

## 总结

根据以上代码的分析，可以看出，`sync.Mutex` 这把锁在你的工作负载（所需时间）比较低，比如只是对某个关键变量赋值的时候，性能还是比较好的，但是如果说对于临界资源的操作耗时很长（特别是单个操作就大于 1ms）的话，实际上性能上会有一定的问题，这也就是我们经常看到 “的锁一直处于饥饿状态” 的问题，对于这种情况，可能就需要另寻他法了。

好了，至此整个 `sync.Mutex` 的分析就此结束了，虽然只有短短 200 行代码（包括 150 行注释，实际代码估计就 50 行），但是其中的算法、设计的思想、编程的理念却是值得感悟，所谓大道至简、少即是多可能就是如此吧。