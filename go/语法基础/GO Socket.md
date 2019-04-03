# GO Socket

## 概念

首先什么事Socket，翻译过来就是孔或者插座。网络上的两个程序通过一个双向的通信连接实现数据的交换，这个连接的一端称为一个socket。
Socket的本质其实是编程接口，是一个IPC接口。（IPC：进程间通信）与其他IPC方法不同的是，它可以通过网络让多个进程建立通信，是的通信双方是否在同一个机器上变得无关紧要。

## Socket如何通信

socket是通过TCP/IP协议族来提供网络链接。Socket是应用程序和运输层之间的抽象层，封装了TCP/IP协议族，用一组简单的接口就能就能通过网络链接通信。下图为网上经典图，用户不需要知道TCP/IP的各种复杂功能协议等，直接使用Socket提供的接口就能完成所有工作。
![图片描述](https://segmentfault.com/img/bVt7xq)

#### Socket通信流程

- 服务端： 首先服务端需要初始化Socket，然后与端口绑定(bind)，对端口进行监听(listen)，调用accept阻塞，等待客户端连接。在这时如果有个客户端初始化一个Socket，然后连接服务器(connect)，如果连接成功，这时客户端与服务器端的连接就建立了。
- 客户端：客户端发送数据请求，服务器端接收请求并处理请求，然后把回应数据发送给客户端，客户端读取数据，最后关闭连接，一次交互结束。

![图片描述](https://segmentfault.com/img/bVN3xV?w=478&h=491)

#### Socket提供的主要接口

- 初始化：int socket(int domain, int type, int protocol)
- 绑定：int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
- 监听：listen()
- 接受请求：accept()

具体参数的意义先不展开，我们主要是看go如何操作socket。

## Go 如何操作Socket

上边简单的介绍了Socket的概念，在go语言中我们可以很方便的使用net包来操作。其实go的net包就是对上面的Socket的接口做了再次封装，让我们能很方便的建立Socket连接和使用连接通信。直接上代码

#### 公共函数

```
//公共函数 用来定义Socket类型 ip 端口。
const(
    Server_NetWorkType = "tcp"
    Server_Address = "127.0.0.1:8085"
    Delimiter = '\t'
)

// 往conn中写数据，可以用于客户端传输给服务端， 也可以服务端返回客户端
func Write(conn net.Conn, content string)(int, error){
    var buffer bytes.Buffer
    buffer.WriteString(content)
    buffer.WriteByte(Delimiter)

    return conn.Write(buffer.Bytes())
}

// 从conn中读取字节流，以上面的结束符为标记
func Read(conn net.Conn)(string, error){
    readBytes := make([]byte,1)
    var buffer bytes.Buffer
    for{
        if _,err := conn.Read(readBytes);err != nil{
            return "", err
        }
        readByte := readBytes[0]
        if readByte == Delimiter{
            break
        }
        buffer.WriteByte(readByte)
    }

    return buffer.String(), nil
}
```

#### **服务端**：

```
func main() {
    // net listen 函数 传入socket类型和ip端口，返回监听对象
    listener, err := net.Listen(socket.Server_NetWorkType,socket.Server_Address)
    if err == nil{
        // 循环等待客户端访问
        for{
            conn,err := listener.Accept()
            if err == nil{
                // 一旦有外部请求，并且没有错误 直接开启异步执行
                go handleConn(conn)
            }
        }
    }else{
        fmt.Println("server error", err)
    }
    defer listener.Close()
}

func handleConn(conn net.Conn){
    for {
        // 设置读取超时时间
        conn.SetReadDeadline(time.Now().Add(time.Second * 2))
        // 调用公用方法read 获取客户端传过来的消息。
        if str, err := socket.Read(conn); err == nil{
            fmt.Println("client:",conn.RemoteAddr(),str)
            // 通过write 方法往客户端传递一个消息
            socket.Write(conn,"server got:"+str)
        }


    }
}
```

#### 客户端

```
func main() {
    // 调用net包中的dial 传入ip 端口 进行拨号连接，通过三次握手之后获取到conn
    conn,err := net.Dial(socket.Server_NetWorkType, socket.Server_Address)
    if err != nil{
        fmt.Println("Client create conn error err:", err)
    }
    defer conn.Close()
    //往服务端传递消息
    socket.Write(conn,"aaaa")
    //读取服务端返回的消息
    if str, err := socket.Read(conn);err == nil{
        fmt.Println(str)
    }

}
```

可以看到，上边的代码很简单。使用net包就可以很轻松的实现Socket通信。

## 简单源码查看

我们可以看到最上边我们介绍的Socket最少需要有创建（socket函数） 绑定（bind函数）监听（listen函数）这些最基本的步骤，这些步骤其实都封装在我们的net包中，到了我们代码中客户从net.Listen 函数里查看源代码。因为代码调用过多只贴一些关键性代码段。
首先 listen 会判断是监听tcp，还是unix。之后经过一些列的调用走到sysSocket方法，这个方法会调用系统的socket方法初始化socket对象返回一个socket的标识符。之后就会使用这个标识符进行绑定 监听。最终返回listener对象。

```go
func Listen(network, address string) (Listener, error) {
    addrs, err := DefaultResolver.resolveAddrList(context.Background(), "listen", network, address, nil)
    if err != nil {
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: nil, Err: err}
    }
    var l Listener
    switch la := addrs.first(isIPv4).(type) {
    case *TCPAddr:
        // 监听TCP 
        l, err = ListenTCP(network, la)
    case *UnixAddr:
        l, err = ListenUnix(network, la)
    default:
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: la, Err: &AddrError{Err: "unexpected address type", Addr: address}}
    }
    if err != nil {
        return nil, err // l is non-nil interface containing nil pointer
    }
    return l, nil
}
// 最终调用的系统方法， 是不是跟socket 的初始化方法很像？
func sysSocket(family, sotype, proto int) (int, error) {
    // See ../syscall/exec_unix.go for description of ForkLock.
    syscall.ForkLock.RLock()
    s, err := socketFunc(family, sotype, proto)
    if err == nil {
        syscall.CloseOnExec(s)
    }
    syscall.ForkLock.RUnlock()
    if err != nil {
        return -1, os.NewSyscallError("socket", err)
    }
    if err = syscall.SetNonblock(s, true); err != nil {
        poll.CloseFunc(s)
        return -1, os.NewSyscallError("setnonblock", err)
    }
    return s, nil
}
```