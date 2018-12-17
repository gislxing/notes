# 使用WebAssembly和Go编写前端Web框架

JavaScript前端框架毫无疑问有助于突破以前在浏览器环境中可能实现的界限。更复杂的应用程序已经建立在React，Angular和VueJS之类的基础之上，仅举几例，并且有一个众所周知的笑话，关于新的前端框架似乎每天都会出现。

然而，这种发展速度对全世界的开发者来说都是一个非常好的消息。通过每个新框架，我们可以发现处理状态的更好方法，或者使用shadow DOM等方式有效地渲染。

然而，最新的趋势似乎是在用JavaScript以外的语言编写这些框架并将它们编译成WebAssembly。我们开始看到由于[Lin Clark](https://twitter.com/linclark)之类的人对JavaScript和WebAssembly进行通信的方式有了重大改进，我们无疑会看到更多重大改进，因为WebAssembly开始在我们的生活中变得更加突出。

# 介绍

因此，在本教程中，我认为构建一个用Go编写的非常简单的前端框架的基础是一个好主意，它编译成WebAssembly。这至少包括以下功能：

- 功能注册
- 组件
- 超简单路由

我现在警告你，虽然这些将非常简单，并且无法准备好生产。如果这篇文章有点受欢迎，那么我希望能够将它推向前进，并尝试构建满足半正式前端框架要求的东西。

> **Github：**这个项目的完整源代码可以在这里找到：[elliotforbes / oak](https://github.com/elliotforbes/oak)。如果您喜欢为项目做出贡献，请随意，我很乐意接受任何拉动请求！

# 初始点

好的，让我们深入选择我们的编辑器并开始编码！我们要做的第一件事就是创建一个非常简单的东西`index.html`，作为我们前端框架的入口点：

```html
<!doctype html>
<!--
Copyright 2018 The Go Authors. All rights reserved.
Use of this source code is governed by a BSD-style
license that can be found in the LICENSE file.
-->
<html>

<head>
	<meta charset="utf-8">
	<title>Go wasm</title>
	<script src="./static/wasm_exec.js"></script>
	<script src="./static/entrypoint.js"></script>
</head>
<body>	

  <div class="container">
    <h2>Oak WebAssembly Framework</h2>
  </div>
</body>

</html>
```

您会注意到这些`js`文件在顶部导入了2个文件，这使我们可以执行完成的WebAssembly二进制文件。第一个是大约414行，所以，为了保持本教程的可读性，我建议你从这里下载它：[https](https://github.com/elliotforbes/oak/blob/master/examples/blog/static/wasm_exec.js)：[//github.com/elliotforbes/oak/blob/master/examples/blog/static/ wasm_exec.js](https://github.com/elliotforbes/oak/blob/master/examples/blog/static/wasm_exec.js)

第二个是我们的`entrypoint.js`档案。这将获取并运行`lib.wasm`我们即将构建的内容。

```js
// static/entrypoint.js
const go = new Go();
WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then((result) => {
    go.run(result.instance);
});
```

最后，既然我们已经解决了这个问题，我们就可以开始深入了解一些Go代码！创建一个新文件`main.go`，其中包含Oak Web Framework的入口点！

```go
// main.go
package main

func main() {
	println("Oak Framework Initialized")
}
```

这很简单。我们创建了一个非常简单的Go程序，只需`Oak Framework Initialized`打开我们的Web应用程序即可打印出来。要验证一切正常，我们需要使用以下命令编译它：

```s
$ GOOS=js GOARCH=wasm go build -o lib.wasm main.go
```

这应该构建我们的Go代码并输出我们在`lib.wasm`文件中引用的`entrypoint.js`文件。

真棒，如果一切正常，那么我们准备在浏览器中试用它！我们可以使用这样一个非常简单的文件服务器：

```go
// server.go
package main

import (
	"flag"
	"log"
	"net/http"
)

var (
	listen = flag.String("listen", ":8080", "listen address")
	dir    = flag.String("dir", ".", "directory to serve")
)

func main() {
	flag.Parse()
	log.Printf("listening on %q...", *listen)
	log.Fatal(http.ListenAndServe(*listen, http.FileServer(http.Dir(*dir))))
}
```

然后，您可以通过键入满足您的应用程序`go run server.go`，你应该能够从访问你的应用程序`http://localhost:8080`。

# 功能注册

好吧，所以我们有一个相当基本的打印声明工作，但在宏观方案中，我不认为它只是作为Web框架的资格。

让我们来看看如何在Go中构建函数并注册这些函数，以便我们可以在我们的函数中调用它们`index.html`。我们将创建一个新的实用程序函数，它将同时包含`string`我们函数的名称以及它将映射到的Go函数。

将以下内容添加到现有`main.go`文件中：

```go
// main.go
import "syscall/js"

// RegisterFunction
func RegisterFunction(funcName string, myfunc func(i []js.Value)) {
	js.Global().Set(funcName, js.NewCallback(myfunc))
}
```

所以，这就是事情开始变得有用的地方。我们的框架现在允许我们注册函数，以便框架的用户可以开始创建自己的功能。

使用我们框架的其他项目可以开始注册自己的函数，这些函数随后可以在他们自己的前端应用程序中使用。

# 组件

所以，我想接下来我们需要考虑添加到我们的框架中的是组件的概念。基本上，我希望能够`components/`在项目中定义一个使用它的目录，并且在该目录中我希望能够构建一个`home.go`具有我的主页所需的所有代码的组件。

那么，我们该怎么做呢？

好吧，React倾向于提供具有`render()`函数的类，这些函数返回HTML / JSX /您希望为所述组件呈现的任何代码。让我们窃取它并在我们自己的组件中使用它。

我本质上希望能够在使用此框架的项目中执行此类操作：

```go
package components

type HomeComponent struct{}

var Home HomeComponent

func (h HomeComponent) Render() string {
	return "<h2>Home Component</h2>"
}
```

因此，在我的`components`包中，我定义了`HomeComponent`一个`Render()`返回HTML 的方法。

为了向我们的框架添加组件，我们将保持简单，并且只需定义`interface`我们随后定义的任何组件必须遵守的组件。`components/comopnent.go`在我们的Oak框架中创建一个新文件：

```go
// components/component.go
package component

type Component interface {
	Render() string
}
```

如果我们想要为各种组件添加新功能会发生什么？好吧，这让我们可以做到这一点。我们可以`oak.RegisterFunction`在`init`组件的功能中使用调用来注册我们想要在组件中使用的任何函数！

```go
package components

import (
	"syscall/js"

	"github.com/elliotforbes/oak"
)

type AboutComponent struct{}

var About AboutComponent

func init() {
	oak.RegisterFunction("coolFunc", CoolFunc)
}

func CoolFunc(i []js.Value) {
	println("does stuff")
}

func (a AboutComponent) Render() string {
	return `<div>
						<h2>About Component Actually Works</h2>
						<button onClick="coolFunc();">Cool Func</button>
					</div>`
}
```

当我们将它与路由器结合起来时，我们应该能够看到我们`HTML`被渲染到我们的页面，我们应该能够点击那个调用的按钮，`coolFunc()`它将`does stuff`在我们的浏览器控制台中打印出来！

太棒了，让我们看看我们现在如何构建一个简单的路由器。

# 构建路由器

好的，我们已经`components`在我们的Web框架中得到了概念。我们差不多完成了吗？

不完全是，我们可能需要的下一件事是在不同组件之间导航的方法。大多数框架似乎都`<div>`具有特定的特性`id`，它们会绑定并呈现其中的所有组件，因此我们将在Oak中窃取相同的策略。

让我们`router/router.go`在我们的橡木框架中创建一个文件，以便我们可以开始破解。

在这个中，我们想要将`string`路径映射到组件，我们不会进行任何URL检查，我们现在只需将所有内容保存在内存中以保持简单：

```go
// router/router.go
package router

import (
	"syscall/js"

	"github.com/elliotforbes/oak/component"
)

type Router struct {
	Routes map[string]component.Component
}

var router Router

func init() {
	router.Routes = make(map[string]component.Component)
}
```

因此，在此范围内，我们创建了一个新`Router`结构，其中包含`Routes`了我们在上一节中定义的组件的字符串映射。

路由不是我们框架中的强制性概念，我们希望用户在他们希望初始化新路由器时进行选择。因此，让我们创建一个新函数，它将注册一个`Link`函数，并将我们映射中的第一个路径绑定到我们的`<div id="view"/>`html标记：

```go
// router/router.go
// ...
func NewRouter() {
	js.Global().Set("Link", js.NewCallback(Link))
	js.Global().Get("document").Call("getElementById", "view").Set("innerHTML", "")
}

func RegisterRoute(path string, component component.Component) {
	router.Routes[path] = component
}

func Link(i []js.Value) {
	println("Link Hit")

	comp := router.Routes[i[0].String()]
	html := comp.Render()

	js.Global().Get("document").Call("getElementById", "view").Set("innerHTML", html)
}
```

您应该注意到，我们已经创建了一个`RegisterRoute`函数，允许我们将a注册`path`到给定的组件。

我们的`Link`功能也非常酷，因为它允许我们在项目中的各个组件之间导航。我们可以指定非常简单的`<button>`元素，以允许我们导航到已注册的路径，如下所示：

```html
<button onClick="Link('link')">Clicking this will render our mapped Link component</button>
```

很棒，所以我们现在有一个非常简单的路由器，如果我们想在一个简单的应用程序中使用它，我们可以这样做：

```go
// my-project/main.go
package main

import (
	"github.com/elliotforbes/oak"
	"github.com/elliotforbes/oak/examples/blog/components"
	"github.com/elliotforbes/oak/router"
)

func main() {
	// Starts the Oak framework
	oak.Start()

	// Starts our Router
	router.NewRouter()
	router.RegisterRoute("home", components.Home)
	router.RegisterRoute("about", components.About)

	// keeps our app running
	done := make(chan struct{}, 0)
	<-done
}
```

# 一个完整的例子

将所有这些放在一起，我们可以开始构建具有组件和路由功能的非常简单的Web应用程序。如果你想看一些关于它是如何工作的例子，那么看一下官方回购中的例子：[elliotforbes / oak / examples](https://github.com/elliotforbes/oak/tree/master/examples)

# 挑战前进

这个框架中的代码绝不是生产准备好的，但是我希望这篇文章能够开始讨论如何在Go中开始构建更多生产就绪的框架。

如果不出意外，它开始了识别仍需要做什么的旅程，以使其成为React / Angular / VueJS之类的可行替代方案，所有这些都是大规模加速开发人员生产力的现象框架。

我希望这篇文章能激励你们中的一些人开始研究如何在这个非常简单的起点上进行改进。

# 结论

如果你喜欢这个教程，那么请随时分享给你的朋友，你的推特或任何你想要的地方，这真的有助于网站，并直接支持我写更多！

我也在YouTube上，所以请随时订阅我的频道以获取更多Go内容！- [TutorialEdge](https://youtube.com/tutorialedge)。