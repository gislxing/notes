# Go 文件上传
## Sending big file with minimal memory in Go Go客户端使用小内存上传大文件

![img](img/1_XDleIjprsIV4KkaRCQdeVQ.jpeg)

​                                                                                      饥饿的地鼠

Most common way of uploading file(s) by http is splitting them in multiple parts (multipart/form-data), this structure helps greatly as we can also attach fields along and send them all in one single request.

通过http上传文件的最常见方式是将它们分成多个部分（multipart / form-data），这种结构非常有用，因为我们还可以在一个请求中附加字段并将它们全部发送出去。

A typical multipart request (example from Mozilla):

典型的多部分请求（来自Mozilla的示例）：

```html
POST /foo HTTP/1.1
Content-Length: 68137
Content-Type: multipart/form-data; boundary=---------------------------974767299852498929531610575
-----------------------------974767299852498929531610575
Content-Disposition: form-data; name="description"
some text
-----------------------------974767299852498929531610575
Content-Disposition: form-data; name="myFile"; filename="foo.txt" 
Content-Type: text/plain
(content of the uploaded file foo.txt)
-----------------------------974767299852498929531610575--
```

We will start with this simple implementation, standard package `mime/multipart` got our back:

我们将从这个简单的实现开始，标准包`mime/multipart`得到了我们的回报：

```
buf := new(bytes.Buffer)
writer := multipart.NewWriter(buf)
defer writer.Close()
part, err := writer.CreateFormFile("myFile", "foo.txt")
if err != nil {
    return err
}
file, err := os.Open(name)
if err != nil {
    return err
}
defer file.Close()
if _, err = io.Copy(part, file); err != nil {
    return err
}
http.Post(url, writer.FormDataContentType(), buf)
```

`multipart.Writer` will automatically enclose the parts (files, fields) in boundary markup before sending them ❤️ We don’t need to get our hands dirty.

`multipart.Writer` 在发送它们之前会自动将部件（文件，字段）封装在边界标记中❤️我们不需要弄脏手。

------

Above works great until you do some benchmark and see the memory allocation **grows linearly with the file size**, so what went wrong? It turns out that `buf` is fully buffered the whole file content, `buf` sequentially reads a modest 32kB from the file but won’t stop until it reaches EOF, so to hold the file content `buf` needs to be at least at the file size, plus some additional boundary markup.

上面的工作很棒，直到你做一些基准测试并看到内存分配**随文件大小线性增长**，所以出了什么问题？事实证明，`buf`整个文件内容是完全缓冲的，`buf`从文件中顺序读取一个32kB，但是直到达到EOF才会停止，所以要保持文件内容`buf`至少需要文件大小，加上一些额外的边界标记。

HTTP/1.1 has a method of transfer data in chunks unboundedly, without specifying `Content-Length`of request body, this is an important feature we can utilize.

HTTP / 1.1有一种无限制地传输数据的方法，没有指定`Content-Length`请求体，这是我们可以利用的一个重要特性。

So `buf` causes problem, but how can we synchronize file content to the request without it? `io.Pipe` is born for these tasks, as the name suggests, it pipes writer to reader:

因此`buf`导致问题，但是如何在没有它的情况下将文件内容与请求同步？`io.Pipe`为这些任务而生，正如其名称所示，它将作家与读者联系起来：

```
r, w := io.Pipe()
m := multipart.NewWriter(w)
go func() {
    defer w.Close()
    defer m.Close()
    part, err := m.CreateFormFile("myFile", "foo.txt")
    if err != nil {
        return
    }
    file, err := os.Open(name)
    if err != nil {
        return
    }
    defer file.Close()
    if _, err = io.Copy(part, file); err != nil {
        return
    }
}()
http.Post(url, m.FormDataContentType(), r)
```

If you dump the request above, the header reads:

如果转储上面的请求，标题为：

```
POST / HTTP/1.1
...
Transfer-Encoding: chunked
Accept-Encoding: gzip
Content-Type: multipart/form-data; boundary=....
User-Agent: Go-http-client/1.1
```

Yep, `net/http` has handled the `Transfer-Encoding` and remove `Content-Length` without us manually doing anything. Wanna see the difference in memory allocation?

是的，`net/http`已处理`Transfer-Encoding`和删除`Content-Length`没有我们手动做任何事情。想看看内存分配的差异吗？

```
Benchmark sending 16MB file
33471060 B/op
84767 B/op
```

Guess which one is the second approach? 😉

## Don’t parse everything from client multipart POST  服务器端处理文件上传

![img](img/1_4Iz1QACB-ebYVS0ALK4zFQ.jpeg)

​                                                                                     I’m not a hoarder

Q: What’s **wrong** with the code below (this is very typical way of parsing a multipart request):

请问：下面的代码有什么**问题**（这是解析多部分请求的非常典型的方式）：

```go
// in the body of a http.HandlerFunc
err := r.ParseMultipartForm(32 << 20) // 32Mb
if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
}
```

A: Nothing wrong with the code above, it’s a hidden gotcha which is very easy to be oversight though.

答：上面的代码没有错，它是一个隐藏的陷阱，虽然很容易被忽视。

`ParseMultipartForm` will parse every single of part of request, if you send 100 files in a “chunked” encoding POST to the above endpoint, it will parse them all. Notice that 32Mb is the bytes allocated to the request body to store in memory, not the limit of request body, when full (says 33Mb), it will write to temporary directory.

`ParseMultipartForm`将解析请求的每一部分，如果您将“chunked”编码POST中的100个文件发送到上述端点，它将全部解析它们。请注意，32Mb是分配给请求体的字节存储在内存中，而不是请求体的限制，当满（33Mb）时，它将写入临时目录。

------

A simple triage 一个简单的分类 :

```go
// add this line on top
r.Body = http.MaxBytesReader(w, r.Body, 32<<20+512)
...
```

Above will limit the POST body size to 32.5Mb and throw error if client attempt to send more than that. At least, we don’t have to worry if client maliciously draining server resource.

如果客户端尝试发送超过该值，则上面将POST主体大小限制为32.5Mb并抛出错误。至少，如果客户端恶意耗尽服务器资源，我们不必担心。

But what if we want the full control of data coming in? E.g.: file size limit or file type on specific field. With the standard `ParseMultipartForm`, we probably can’t achieve that.

但是如果我们想要完全控制数据呢？例如：特定字段的文件大小限制或文件类型。有了标准`ParseMultipartForm`，我们可能无法做到这一点。

These are the things I want to have in my handler:

这些是我想在我的处理程序中拥有的东西：

- file type validation 

  文件类型验证

- file size validation 

  文件大小验证

- whitelist “field” names (so we don’t write down any file we don’t want to)

  白名单“字段”名称（所以我们不写下任何我们不想要的文件）

- parse fields in order and terminate if not (this is opinionated, but who wouldn’t prefer bounded and deterministic)

  按顺序解析字段，如果没有则终止（这是自以为是，但谁不喜欢有界和确定性）

- terminate early if any validation fails

  如果任何验证失败，则提前终止

- buffering all the way, no pre-allocation

  一路缓冲，没有预先分配

So this is what I come up with, for demonstration purpose, POST will have 2 fields named: `text_field` and `file_field`, bear with me for below extremely verbose code:

所以这就是我想出来的，出于演示目的，POST将有两个名为的字段：`text_field`并且`file_field`，请耐心等待以下非常详细的代码：

```go
// function body of a http.HandlerFunc
r.Body = http.MaxBytesReader(w, r.Body, 32<<20+1024)
reader, err := r.MultipartReader()
if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
}
// parse text field
text := make([]byte, 512)
p, err := reader.NextPart()
// one more field to parse, EOF is considered as failure here
if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
if p.FormName() != "text_field" {
    http.Error(w, "text_field is expected", http.StatusBadRequest)
    return
}
_, err = p.Read(text)
if err != nil && err != io.EOF {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
// parse file field
p, err = reader.NextPart()
if err != nil && err != io.EOF {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
if p.FormName() != "file_field" {
    http.Error(w, "file_field is expected", http.StatusBadRequest)
    return
}
buf := bufio.NewReader(p)
sniff, _ := buf.Peek(512)
contentType := http.DetectContentType(sniff)
if contentType != "application/zip" {
    http.Error(w, "file type not allowed", http.StatusBadRequest)
    return
}
f, err := ioutil.TempFile("", "")
if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
defer f.Close()
var maxSize int64 = 32 << 20
lmt := io.LimitReader(p, maxSize+1)
written, err := io.Copy(f, lmt)
if err != nil && err != io.EOF {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
if written > maxSize {
    os.Remove(f.Name())
    http.Error(w, "file size over limit", http.StatusBadRequest)
    return
}
// schedule for other stuffs (s3, scanning, etc.)
```

Some key points for the code above:

上面代码的一些关键点：

- Only fetch first 2 parts in POST body, handler will expect `text_field` then `file_field`

  只在POST主体中获取前2个部分，`text_field`然后处理程序会期望`file_field`

- Assert file type using `bufio.Reader` wrapping Part reader (peeking first 512 bytes)

  使用`bufio.Reader`包装部件读取器（先查看512字节）断言文件类型

- Limit file size with `io.LimitReader` (with 1 byte offset to see if part reader still has some data left)

  限制文件大小`io.LimitReader`（使用1个字节的偏移量来查看部分阅读器是否还有一些数据）

The code can be definitely refactored to treat file and text fields in general but you got the idea, let me know what you think in the comment section.

一般来说代码可以被重构来处理文件和文本字段，但你明白了，让我知道你在评论部分的想法

Happy coding gophers!