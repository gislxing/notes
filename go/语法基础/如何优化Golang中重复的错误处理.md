# 如何优化Golang中重复的错误处理

Golang 错误处理最让人头疼的问题就是代码里充斥着「if err != nil」，它们破坏了代码的可读性，本文收集了几个例子，让大家明白如何优化此类问题。



让我们看看 [Errors are values](https://blog.golang.org/errors-are-values) 中提到的一个 io.Writer 例子：

```
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
```

如上代码里充斥着大量重复的「if err != nil」错误判断，让人无法直观的看出代码本来的意图是什么，看上去是坏味道，改进版：

```
type errWriter struct {
	w   io.Writer
	err error
}

func (ew *errWriter) write(buf []byte) {
	if ew.err != nil {
		return
	}
	_, ew.err = ew.w.Write(buf)
}

ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
if ew.err != nil {
    return ew.err
}
```

通过自定义类型 errWriter 来封装 io.Writer，并且封装了 error，新类型有一个 write 方法，不过其方法签名并没有返回 Error，而是在方法内部判断一旦有问题就立刻返回，有了这些准备工作，我们就可以把原本穿插在业务逻辑中间的错误判断提出来放到最后来统一调用，从而在视觉上保证让人可以直观的看出代码本来的意图是什么。

让我们再看看 [Eliminate error handling by eliminating errors](https://dave.cheney.net/2019/01/27/eliminate-error-handling-by-eliminating-errors) 中提到的另一个 io.Writer 例子：

```
type Header struct {
	Key, Value string
}

type Status struct {
	Code   int
	Reason string
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
	_, err := fmt.Fprintf(w, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
	if err != nil {
		return err
	}

	for _, h := range headers {
		_, err := fmt.Fprintf(w, "%s: %s\r\n", h.Key, h.Value)
		if err != nil {
			return err
		}
	}

	if _, err := fmt.Fprint(w, "\r\n"); err != nil {
		return err
	}

	_, err = io.Copy(w, body)
	return err
}
```

乍一看错误是 fmt.Fprint 和 io.Copy 返回的，难道我们要重新封装一下它们？实际上真正的源头是它们的参数 io.Writer，因为直接调用 io.Writer 的 Writer 方法的话，方法签名中有返回值 error，所以每一步 fmt.Fprint 和 io.Copy 操作都不得不进行重复的错误处理，看上去是坏味道，改进版：

```
type errWriter struct {
	io.Writer
	err error
}

func (e *errWriter) Write(buf []byte) (int, error) {
	if e.err != nil {
		return 0, e.err
	}

	var n int
	n, e.err = e.Writer.Write(buf)
	return n, nil
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
	ew := &errWriter{Writer: w}
	fmt.Fprintf(ew, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)

	for _, h := range headers {
		fmt.Fprintf(ew, "%s: %s\r\n", h.Key, h.Value)
	}

	fmt.Fprint(ew, "\r\n")
	io.Copy(ew, body)

	return ew.err
}
```

通过自定义类型 errWriter 来封装 io.Writer，并且封装了 error，同时重写了 Writer 方法，虽然方法签名中仍然有返回值 error，但是我们单独保存了一份 error，并且在方法内部判断一旦有问题就立刻返回，有了这些准备工作，新版的 WriteResponse 不再有重复的错误判断，只需要在最后检查一下 Error 即可。

类似的做法在 Golang 标准库中屡见不鲜，让我们继续看看 [Eliminate error handling by eliminating errors](https://dave.cheney.net/2019/01/27/eliminate-error-handling-by-eliminating-errors) 中提到的一个关于 bufio.Reader 和 bufio.Scanner 的例子：

```
func CountLines(r io.Reader) (int, error) {
	var (
		br    = bufio.NewReader(r)
		lines int
		err   error
	)

	for {
		_, err = br.ReadString('\n')
		lines++
		if err != nil {
			break
		}
	}

	if err != io.EOF {
		return 0, err
	}

	return lines, nil
}
```

我们构造一个 bufio.Reader，然后在一个循环中调用 ReadString 方法，如果读到文件结尾，那么 ReadString 会返回一个错误（io.EOF），为了判断此类情况，我们不得不在每次循环时判断「if err != nil」，看上去这是坏味道，改进版：

```
func CountLines(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	lines := 0

	for sc.Scan() {
		lines++
	}

	return lines, sc.Err()
}
```

实际上，和 bufio.Reader 相比，bufio.Scanner 是一个更高阶的类型，换句话简单点来说的话，相当于是 bufio.Scanner 抽象了 bufio.Reader，通过把低阶的 bufio.Reader 换成高阶的 bufio.Scanner，循环中不再需要判断「if err != nil」，因为 Scan 方法签名不再返回 error，而是返回 bool，当在循环里读到了文件结尾的时候，循环直接结束，如此一来，我们就可以统一在最后调用 Err 方法来判断成功还是失败，看看 Scanner 的定义：

```
type Scanner struct {
	r            io.Reader // The reader provided by the client.
	split        SplitFunc // The function to split the tokens.
	maxTokenSize int       // Maximum size of a token; modified by tests.
	token        []byte    // Last token returned by split.
	buf          []byte    // Buffer used as argument to split.
	start        int       // First non-processed byte in buf.
	end          int       // End of data in buf.
	err          error     // Sticky error.
	empties      int       // Count of successive empty tokens.
	scanCalled   bool      // Scan has been called; buffer is in use.
	done         bool      // Scan has finished.
}
```

可见 Scanner 封装了 io.Reader，并且封装了 error，和我们之前讨论的做法一致。有一点说明一下，实际上查看 Scan 源代码的话，你会发现它不是通过 err 来判断是否结束的，而是通过 done 来判断是否结束，这是因为 Scan 只有遇到文件结束的错误才退出，其它错误会继续执行，当然，这只是具体的细节问题，不影响我们的结论。

通过对以上几个例子的分析，我们可以得出优化重复错误处理的方法：通过创建新的类型来封装原本干脏活累活的旧类型，同时在新类型中封装 error，至于具体的逻辑实现，先判断有没有 error，如果有就直接退出，如果没有就继续执行，并且在执行过程中保存可能出现的 error，最后通过统一调用新类型的 error 来完成错误处理。提醒一下，此方案的缺点是要到最后才能知道有没有错误，好在如此的控制粒度在多数时候并无大碍。