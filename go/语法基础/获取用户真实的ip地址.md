# 获取用户真实的ip地址

用户请求到达提供服务的服务器中间有很多的环节，导致服务获取用户真实的 ip 非常困难，大多数的框架及工具库都会封装各种获取用户真实 ip 的方法，在[exnet](https://github.com/thinkeridea/go-extend/blob/master/exnet) 包中也封装了各种 ip 相关的操作，其中就包含获取客户端 ip 的方法，比较实用的方法如下：

- `func ClientIP(r *http.Request) string` ClientIP 尽最大努力实现获取客户端 IP 的算法。 解析 X-Real-IP 和 X-Forwarded-For 以便于反向代理（nginx 或 haproxy）可以正常工作。
- `func ClientPublicIP(r *http.Request) string` ClientPublicIP 尽最大努力实现获取客户端公网 IP 的算法。 解析 X-Real-IP 和 X-Forwarded-For 以便于反向代理（nginx 或 haproxy）可以正常工作。
- `func HasLocalIP(ip net.IP) bool` HasLocalIP 检测 IP 地址是否是内网地址
- `func HasLocalIPddr(ip string) bool` HasLocalIPddr 检测 IP 地址字符串是否是内网地址
- `func RemoteIP(r *http.Request) string` RemoteIP 通过 RemoteAddr 获取 IP 地址， 只是一个快速解析方法。



## 获取用户真实ip地址

`ClientIP` 方法 与 `ClientPublicIP` 方法的实现类似，只是一个按照 http 协议约定获取客户端 ip， 一个按照约定格式查找到公网 ip。

在网络与服务架构、业务逻辑复杂的环境中，按照 http 协议约定的方式，并非总能获取到真实的 ip，在我们的业务中用户流量经由三方多层级转发(都是三方自己实现的http client) ，难免会出现一些纰漏，这时越往后的服务获取用户真实 ip 越加困难，你甚至不知道自己获取的 ip 是否是真实的。

但是我们的客户经由三方转发而来的流量，那么客户极大多数甚至排除测试之外都是公网用户，结合使用 `ClientPublicIP` 和 `ClientIP` 方法总能更好的获取用户的真实 ip。

```
// var r *http.Request
ip := exnet.ClientPublicIP(r)
if ip == ""{
  ip = exnet.ClientIP(r)
}
```

用上面的方法总能有效的获取用户真实的 ip 地址，下面分析下两个方法的具体实现。

```
// ClientIP 尽最大努力实现获取客户端 IP 的算法。
// 解析 X-Real-IP 和 X-Forwarded-For 以便于反向代理（nginx 或 haproxy）可以正常工作。
func ClientIP(r *http.Request) string {
	xForwardedFor := r.Header.Get("X-Forwarded-For")
	ip := strings.TrimSpace(strings.Split(xForwardedFor, ",")[0])
	if ip != "" {
		return ip
	}

	ip = strings.TrimSpace(r.Header.Get("X-Real-Ip"))
	if ip != "" {
		return ip
	}

	if ip, _, err := net.SplitHostPort(strings.TrimSpace(r.RemoteAddr)); err == nil {
		return ip
	}

	return ""
}
```

`ClientIP` 首先读取 `X-Forwarded-For` header 中用 `,` 分隔的第一个ip地址，如果这个地址不存在，就会从 `X-Real-Ip` header 中获取，如果还是不存在，说明流量并非是由反向代理转发而来，而是客户端直接请求服务，这时通过 `http.Request.RemoteAddr` 字段截取除去端口号的 ip 地址。

这个方法很简单，就是按照 http 约定的格式获取，其中 `X-Forwarded-For` 和 `X-Real-Ip` header 由反向代理填充，例如 nginx 或 haproxy。

```
// ClientPublicIP 尽最大努力实现获取客户端公网 IP 的算法。
// 解析 X-Real-IP 和 X-Forwarded-For 以便于反向代理（nginx 或 haproxy）可以正常工作。
func ClientPublicIP(r *http.Request) string {
	var ip string
	for _, ip = range strings.Split(r.Header.Get("X-Forwarded-For"), ",") {
		ip = strings.TrimSpace(ip)
		if ip != "" && !HasLocalIPddr(ip) {
			return ip
		}
	}

	ip = strings.TrimSpace(r.Header.Get("X-Real-Ip"))
	if ip != "" && !HasLocalIPddr(ip) {
		return ip
	}

	if ip, _, err := net.SplitHostPort(strings.TrimSpace(r.RemoteAddr)); err == nil {
		if !HasLocalIPddr(ip) {
			return ip
		}
	}

	return ""
}
```

`ClientPublicIP` 很简单，和 `ClientIP` 方法的读取顺序一样，只是试图中 `X-Forwarded-For` 列表中找到一个公网ip，如果没有检查 `X-Real-Ip` 是否是一个公网 ip，其次检查 `http.Request.RemoteAddr` 是否是公网ip，如果没有找到公网 ip 这返回一个空字符串。

这个方法可以让我们有机会优先获取到用户的公网 ip，往往公网 ip 对我们来说更有价值。

## 检查ip对否是内网地址

[exnet](https://github.com/thinkeridea/go-extend/blob/master/exnet) 中还提供了检查 ip 地址是否是内网地址，这在有些情况下非常有用，比如：服务中有些接口只能内网访问，也就是只允许管理员访问（例如动态设定日志级别、查看服务 pprof 信息）；我们想隐藏后端服务，只暴露给用户负载均衡(反向代理)，用户无法直接访问我们的服务，这些方法及其有用，下面看看具体实现。

我的服务提供了动态设置日志级别，以便服务出现问题，可以第一时间查看调试日志分析具体原因，但是这个接口很危险，不应该暴露给公网，所以会用路由中间件检查请求是否来自公网，来自公网则返回 404。

该方法认为如下地址段都是内网地址：

```
10.0.0.0/8
169.254.0.0/16
172.16.0.0/12
172.17.0.0/12
172.18.0.0/12
172.19.0.0/12
172.20.0.0/12
172.21.0.0/12
172.22.0.0/12
172.23.0.0/12
172.24.0.0/12
172.25.0.0/12
172.26.0.0/12
172.27.0.0/12
172.28.0.0/12
172.29.0.0/12
172.30.0.0/12
172.31.0.0/12
192.168.0.0/16
// HasLocalIPddr 检测 IP 地址字符串是否是内网地址
func HasLocalIPddr(ip string) bool {
	return HasLocalIP(net.ParseIP(ip))
}

// HasLocalIP 检测 IP 地址是否是内网地址
func HasLocalIP(ip net.IP) bool {
	for _, network := range localNetworks {
		if network.Contains(ip) {
			return true
		}
	}

	return ip.IsLoopback()
}
```

两个检查方法实现差异仅接受参数类型不一致，检查过程都是逐个对比内网 ip 段是否包含该ip地址，如果不包含则判断该地址是否是回环地址。

## 获取反向代理ip

如何判断改地址来自反向代理服务器呢，不同的反向代理实现都有些差异，4 层反向代理甚至可以提供用户的真实 ip（`http.Request.RemoteAddr` 是用户的ip，而不是反向代理的）， 而隐藏自己的ip，这里说一下常见的方法。

往往 `http.Request.RemoteAddr` 保存最后一个连接服务的客户端 ip，我们获取反向代理的ip地址，最简单有效的方法就是通过 `http.Request.RemoteAddr` 获取， [exnet](https://github.com/thinkeridea/go-extend/blob/master/exnet) 中提供了 `RemoteIP` 的快捷方法，实现如下：

```
// RemoteIP 通过 RemoteAddr 获取 IP 地址， 只是一个快速解析方法。
func RemoteIP(r *http.Request) string {
	if ip, _, err := net.SplitHostPort(strings.TrimSpace(r.RemoteAddr)); err == nil {
		return ip
	}

	return ""
}
```

这是一个非常方便的脚手架，它仅仅切分 `http.Request.RemoteAddr` 的 ip 和端口，并返回有效的ip地址，但却可以简化我们的编写业务代码。