# Golang实现 Eureka-Client

## 原理

根据Java版本的源码，可以看出client主要是通过REST请求来与server进行通信。

Java版本的核心实现：`com.netflix.discovery.DiscoveryClient`。

其中主要逻辑如下：

- client启动时注册信息到server
- 定时心跳、刷新服务列表，主要是两个线程池：`heartbeatExecutor`、`cacheRefreshExecutor`
- client关闭时删除注册信息



## 实现

这里不限制语言，主要是发送REST请求到server。

### 注册信息

通过POST请求，将服务信息注册到server。

请求地址：`POST /eureka/apps/{APP_NAME}`

信息如下：（不是完整的信息）

```
{
    "instance":{
        "instanceId":"192.168.1.107:golang-example:10000",
        "hostName":"192.168.1.107",
        "ipAddr":"192.168.1.107",
        "app":"golang-example",
        "port":{
            "@enabled":"true",
            "$":10000
        },
        "securePort":{
            "@enabled":"true",
            "$":443
        },
        "status":"UP",
        "overriddenStatus":"UNKNOWN",
        "dataCenterInfo":{
            "name":"MyOwn",
            "@class":"com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo"
        }
    }
}
```

### 定时心跳、刷新服务列表

服务启动后，接下来就是维持client与server之间的心跳等。

#### 定时心跳

默认情况下是30秒发送心跳信息到server。

请求地址：`PUT /eureka/apps/{APP_NAME}/{INSTANCE_ID}?status=UP&lastDirtyTimestamp={TIMESTAMP}`

#### 定时刷新服务列表

默认情况下是30秒刷新服务列表。

刷新服务列表有全量和增量两种方式:

- 全量：`GET /eureka/apps`
- 增量（delta）：`GET /eureka/apps/delta`

其中，全量就是每次都拉取到所有服务信息；而增量拉取变化的服务信息，然后本地去做更新。

为了方便我**只实现了全量拉取**，没有实现delta。

### 删除注册信息

在服务停止时，删除注册的信息即可。

请求地址：`DELETE /eureka/apps/{APP_NAME}/{INSTANCE_ID}`



## Golang核心实现

有2个定时器：

- `refreshTicker`来刷新服务列表
- `heartbeatTicker`进行心跳

```
func (c *Client) Start() {
    c.mutex.Lock()
    c.Running = true
    c.mutex.Unlock()

    refreshTicker := time.NewTicker(c.EurekaClientConfig.RefreshIntervalSeconds)
    heartbeatTicker := time.NewTicker(c.EurekaClientConfig.HeartbeatIntervalSeconds)

    go func() {
        for range refreshTicker.C {
            if c.Running {
                if err := c.doRefresh(); err != nil {
                    fmt.Println(err)
                }
            } else {
                break
            }
        }
    }()

    go func() {
        if err := c.doRegister(); err != nil {
            fmt.Println(err)
        }
        for range heartbeatTicker.C {
            if c.Running {
                if err := c.doHeartbeat(); err != nil {
                    fmt.Println(err)
                }
            } else {
                break
            }
        }
    }()
}
```



## 例子

下面是使用的例子，为了在client停止时删除注册信息，这里用到了`signal`。

```
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    client "github.com/xuanbo/eureka-client"
)

func main() {
    // 1.创建客户端
    c := client.NewClient(&client.EurekaClientConfig{
        DefaultZone: "http://127.0.0.1:8080/eureka/",
        App:         "golang-example",
        Port:        10000,
    })
    // 2.启动client，注册到server。并定心跳、刷新服务列表
    c.Start()

    sigs := make(chan os.Signal)
    exit := make(chan bool, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    // 随便弄一个请求
    http.HandleFunc("/services", func(writer http.ResponseWriter, request *http.Request) {
        // 3.获取所有服务（status均为UP）
        services := c.Services
        
        b, _ := json.Marshal(services)
        _, _ = writer.Write(b)
    })
    server := &http.Server{
        Addr:    ":10000",
        Handler: http.DefaultServeMux,
    }

    // 启动http服务
    go func() {
        if err := server.ListenAndServe(); err != nil {
            fmt.Println(err)
        }
    }()

    // 关闭
    go func() {
        fmt.Println(<-sigs)

        // 停止http服务
        if err := server.Close(); err != nil {
            panic(err)
        }

        // 4.停止客户端，并删除注册信息
        c.Shutdown()

        exit <- true
    }()

    <-exit
}
```

主要是4步，用起来比较简单。



## Github地址

[Github](https://github.com/xuanbo/eureka-client)