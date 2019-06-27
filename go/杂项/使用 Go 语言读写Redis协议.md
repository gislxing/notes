# 使用 Go 语言读写Redis协议

这篇文章使用两个简单的Reader和Writer实现了redis客户端的读写协议，通过这两个实现可以容易地学习Redis协议是如何工作的。

如果你想寻找一个全功能的、产品级的Redis client, 推荐你看看 Gary Burd的 [redigo](https://github.com/gomodule/redigo)

开始之前，建议你先阅读一下 Redis协议的介绍。

官方的协议可以在其网站上找到: [protocol](https://redis.io/topics/protocol)。 Redis的协议叫做 **RESP** (**RE**dis **S**erialization **P**rotocol)，客户端和服务器端通过基于文本的协议进行通讯。

所有的服务器和客户端之间的通讯都使用以下5中基本类型：

- **简单字符串**: 服务器用来返回简单的结果，比如"OK"或者"PONG"
- **bulk string**: 大部分单值命令的返回结果，比如 GET, LPOP, and HGET
- **整数**: 查询长度的命令的返回结果
- **数组**: 可以包含其它RESP对象，设置数组，用来发送命令给服务器，也用来返回多个值的命令
- **Error**: 服务器返回错误信息

RESP的第一个字节表示数据的类型：

- **简单字符串**: 第一个字节是 "+", 比如 "+OK\r\n"
- **bulk string**: 第一个字节是 "$", 比如 "$6\r\nfoobar\r\n"
- **整数**: 第一个字节是 ":"， 比如 ":1000\r\n"
- **数组**: 第一个字节是 "*", 比如 "*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"
- **Error**: 第一个字节是 "-"， 比如 "-Error message\r\n"

基本了解Redis的协议之后，我们就可以实现它的读写器了。



## RESPWriter

假定我们要实现的client很简单，只会发送 bulk string给服务器，下面的代码是它的实现：

```
package redis

import (
  "bufio"
  "io"
  "strconv"     // for converting integers to strings
)

var (
  arrayPrefixSlice      = []byte{'*'}
  bulkStringPrefixSlice = []byte{'$'}
  lineEndingSlice       = []byte{'\r', '\n'}
)

type RESPWriter struct {
  *bufio.Writer
}

func NewRESPWriter(writer io.Writer) *RESPWriter {
  return &RESPWriter{
    Writer: bufio.NewWriter(writer),
  }
}

func (w *RESPWriter) WriteCommand(args ...string) (err error) {
  // 首先写入数组的标志和数组的数量
  w.Write(arrayPrefixSlice)
  w.WriteString(strconv.Itoa(len(args)))
  w.Write(lineEndingSlice)

  // 写入批量字符串
  for _, arg := range args {
    w.Write(bulkStringPrefixSlice)
    w.WriteString(strconv.Itoa(len(arg)))
    w.Write(lineEndingSlice)
    w.WriteString(arg)
    w.Write(lineEndingSlice)
  }

  return w.Flush()
}
```

注意我们并没有直接写入`net.Conn`,而是写入一个`io.Writer`对象中，这样方便我们写测试代码，测试代码中不必以来`net`包。

例如，我们可以使用`bytes.Buffer`来测试我们发送的命令：

```
var buf bytes.Buffer
writer := NewRESPWriter(&buf)
writer.WriteCommand("GET", "foo")
buf.Bytes() // *2\r\n$3\r\nGET\r\n$3\r\nfoo\r\n
```

## RESPReader

客户端使用 `RESPWriter` 来写命令，而`RESPReader`用来读取返回结果， 它尝试从`net.Conn`中一直读取数据，直到读取到一个完整的响应。

需要引入的包:

```
package redis

import (
  "bufio"
  "bytes"
  "errors"
  "io"
  "strconv"
)
```

定义常量和错误：

```
const (
  SIMPLE_STRING = '+'
  BULK_STRING   = '$'
  INTEGER       = ':'
  ARRAY         = '*'
  ERROR         = '-'
)

var (
  ErrInvalidSyntax = errors.New("resp: invalid syntax")
)
```

和`RESPWriter`一样， `RESPReader`并不关心它读取到的对象的细节，它所有的工作就是从连接中读取一个完整的RESP对象。 它需要传入一个`io.Reade`用来读取，并且在内部使用`bufio.Reader`包装了一下。`RESPReader`的定义很简单：

```
type RESPReader struct {
  *bufio.Reader
}

func NewReader(reader io.Reader) *RESPReader {
  return &RESPReader{
    Reader: bufio.NewReaderSize(reader, 32*1024),
  }
}
```

这个缓存大小只是在开发过程中拍脑袋定的，在实际使用中，你可以需要它是可配置的，并且在测试中进行调优。32KB在开发测试中应该没有问题。

`RESPReader`只有一个暴露的方法:`ReadObject()`,它返回这个RESP Object的字节slice。它会返回读取中的错误，以及解析命令的时候的错误。

RESP的第一个字节意味这我们只需读取第一个字节就可以直到如何处理接下来的数据，但是我们总是需要读取一行数据，所以我们还是先读取第一行再处理:

```
func (r *RESPReader) ReadObject() ([]byte, error) {
  line, err := r.readLine()
  if err != nil {
    return nil, err
  }

  switch line[0] {
  case SIMPLE_STRING, INTEGER, ERROR:
    return line, nil
  case BULK_STRING:
    return r.readBulkString(line)
  case ARRAY:
    return r.readArray(line) default:
    return nil, ErrInvalidSyntax
  }
}
```

如果我们读取的这一行是简单字符串、整数或者是Error,我们只需返回这完整的一行就可以了，因为这一行包含了完整的RESP对象。
在`readLine`方法中，我们一直读取直到读取到`\n`,并且检查它前一个字符是否是`\r`, 如果是返回这一行数据 (注意line结尾中包含`\r\n`)：

```
func (r *RESPReader) readLine() (line []byte, err error) {
  line, err = r.ReadBytes('\n')
  if err != nil {
    return nil, err
  }

  if len(line) > 1 && line[len(line)-2] == '\r' {
    return line, nil
  } else {
    // Line was too short or \n wasn't preceded by \r.
    return nil, ErrInvalidSyntax
  }
}
```

接下来我们在看看`readBulkString`的实现。我们需要解析长度值，以便我们接下来决定要读取多少字节。这样我们可以读取**count**字节的数据，并且再读取`\r\n`:

```
func (r *RESPReader) readBulkString(line []byte) ([]byte, error) {
  count, err := r.getCount(line)
  if err != nil {
    return nil, err
  }
  if count == -1 {
    return line, nil
  }

  buf := make([]byte, len(line)+count+2)
  copy(buf, line)
  _, err = io.ReadFull(r, buf[len(line):])
  if err != nil {
    return nil, err
  }

  return buf, nil
}
```

其中`getCount`单独抽取出来了，因为解析数组的时候也需要它：

```
func (r *RESPReader) getCount(line []byte) (int, error) {
  end := bytes.IndexByte(line, '\r')
  return strconv.Atoi(string(line[1:end]))
}
```

为了处理数组，我们首先需要解析数组的数量，然后循环地调用`ReadObject`,将读取到的字节slice放入到结果buffer中:

```
func (r *RESPReader) readArray(line []byte) ([]byte, error) {
  // Get number of array elements.
  count, err := r.getCount(line)
  if err != nil {
    return nil, err
  }

  // Read `count` number of RESP objects in the array.
  for i := 0; i < count; i++ {
    buf, err := r.ReadObject()
    if err != nil {
      return nil, err
    }
    line = append(line, buf...)
  }

  return line, nil
}
```

## 最后总结

上面的百行代码就实现了完整的读写Redis RESP对象，但是在应用到产品环境之前，还有一些东西需要补上：

- 需要从`RESP`对象中读取实际的值。当前`RESPReader`只是返回整个的RESP字节slice,它并没有返回字符串或者整数，当然实现起来也很容易
- `RESPReader`需要更好的错误处理

代码也没进行优化，对于内存分配和复制的优化也没有做，你可以看看成熟的产品级的redis库的实现。