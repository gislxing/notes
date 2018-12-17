# Golang 依赖管理：Go Modules

通常我们会在一个 repo(仓库) 中创建一组 Go package，repo 的路径比如：github.com/myrepo/demo 会作为 go package 的导入路径(import path)，Go 1.11给这样的一组在同一repo下面的packages赋予了一个新的抽象概念: `module`，并启用一个新的文件 `go.mod` 记录 module 的元信息

## 1 概念

一个模块就是把相关联的 Go 包作为单个单元一起版本化的集合。通常，单个版本控制仓库与单个模块是完全对应的，但是单个版本控制仓库也可以包含多个 modules

> Modules 必须是语义化版本，格式：`v<v.m.p>`, 其中主版本号 v 必须存在
>
> 在使用 Git 作为版本控制的项目中，项目发布记录的 tag 号也可以用作它们的版本
>
> 一个 Repo 下可以拥有多个 modules

## 2 go.mod

一个模块是由Go源文件树定义，且在源文件树的根目录下包含了一个 go.mod 文件。模块源代码可能位于 $GOPATH 之外。

> go.mod 文件的作用：记录 module 的元信息，属于文本文件
>
> 使用了 Go Module 后，源码不一定要在 GOPATH 中进行

这里有四个指令可以使用：`module、require、exclude、replace`

> require: 语句指定的依赖项模块
>
> replace: 语句可以替换依赖项模块
>
> exclude: 语句可以忽略依赖项模块

## Go Module 命令：`go mod`

Usage:

```
go mod <command> [<arguments>]
```

The commands are:

```go
download    download modules to local cache             // 下载模块到本地缓存
edit        edit go.mod from tools or scripts           // 从工具或脚本来编辑 go.mod(记录module元信息的文件)
graph       print module requirement graph
init        initialize new module in current directory  // 在当前目录初始化新的 module，会生成 go.mod 文件
tidy        add missing and remove unused modules       // 添加丢失的或移除不再使用的 modules
vendor      make vendored copy of dependencies          // 将依赖包复制到项目下的 vendor 目录
verify      verify dependencies have expected content
why         explain why packages or modules are needed
```

Use “go help mod ” for more information about a command.

Others usefull:

```go
go list -m all          显示依赖关系
go list -m -json all    显示详细的依赖关系
```

## Go Module 初始化

> Go Module 和传统的 GOPATH 不同，不需要包含例如：src、bin 这样的子目录，一个源代码目录甚至是空目录都可以作为 `module`，只要其中包含 go.mod 文件即可。

要初始化 module, 需要使用如下命令:

```bash
$ go mod init [module]
```

当我们使用 `go build`、`go test` 以及 `go list` 时，Go 会自动更新 go.mod 文件，并且将依赖关系写入其中

go 自动查找了依赖并完成下载（国内被墙，可能需要VPN），但是下载的依赖包并不是下载到了 `$GOPATH` 中，而是在 `$GOPATH/pkg/mod` 目录下，且多个项目可以共享缓存的 module

modules 是由Go源文件目录结构定义的，如果目录下含有 `go.mod` 文件，该目录称为模块根目录 `（module root）`，模块根目录及其子目录所有的 Go 包都是属于该 modules 的，但是如果子目录包含有了自己的 `go.mod` 文件，就不再隶属于父 module 下， 而是有了自己的 module



























