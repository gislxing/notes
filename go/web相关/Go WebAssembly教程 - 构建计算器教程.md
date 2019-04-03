# Go WebAssembly教程 - 构建计算器教程

欢迎大家！随着Go v1.11刚刚发布，其中包含了一个包含WebAssembly的实验端口，我认为看看我们如何编写直接编译为WebAssembly的自己的Go程序真是太棒了！

因此，在本文中，我们将构建一个非常简单的计算器，让我们了解如何编写可以暴露给前端的函数，评估DOM元素，然后使用任何结果更新任何DOM元素我们称之为的功能

这将有希望向您展示为前端应用程序编写和编译自己的基于Go的程序所需的内容。

> 如果您还没有从开场线猜到，那么为了使本教程正常运行，需要使用Go v1.11！

# 介绍

那么这对Go和Web开发人员来说意味着什么呢？好吧，它使我们能够使用Go语言编写我们的前端Web应用程序，并随后使用其所有酷炫功能，例如类型安全性，[goroutines](https://tutorialedge.net/golang/concurrency-with-golang-goroutines/)等。

现在，这不是我们第一次看到Go语言用于前端目的。GopherJS已经存在了很长一段时间并且非常成熟，但不同之处在于它将Go代码编译为JS而不是WebAssembly。

# 一个简单的例子

让我们从一个非常简单的示例开始，`Hello World`只要我们单击网页中的按钮，就会在控制台中输出。我知道这听起来令人兴奋，但我们可以很快将其构建成更实用，更酷的东西：

```go
package main

func main() {
	println("Hello World")
}
```

现在，为了编译它，你必须设置`GOARCH=wasm`，`GOOS=js`你还必须使用这样的`-o`标志指定文件的名称，如下所示：

```s
$ GOARCH=wasm GOOS=js go build -o lib.wasm main.go
```

此命令应将我们的代码编译到`lib.wasm`当前工作目录中的文件中。我们将使用该`WebAssembly.instantiateStreaming()`函数将其加载到我们的页面中`index.html`。注意 - 此代码是从官方Go语言回购中窃取的：

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
</head>

<body>

	<script src="wasm_exec.js"></script>

	<script>
		if (!WebAssembly.instantiateStreaming) { // polyfill
			WebAssembly.instantiateStreaming = async (resp, importObject) => {
				const source = await (await resp).arrayBuffer();
				return await WebAssembly.instantiate(source, importObject);
			};
		}

		const go = new Go();
		
		let mod, inst;

		WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then((result) => {
			mod = result.module;
			inst = result.instance;
			document.getElementById("runButton").disabled = false;
		});

		async function run() {
			await go.run(inst);
			inst = await WebAssembly.instantiate(mod, go.importObject); // reset instance
		}

	</script>

	<button onClick="run();" id="runButton" disabled>Run</button>
</body>
</html>
```

我们还需要`wasm_exec.js`可以在[这里](https://github.com/golang/go/blob/master/misc/wasm/wasm_exec.js)找到的文件。下载并将其保存在您的旁边`index.html`。

```html
$ wget https://github.com/golang/go/blob/master/misc/wasm/wasm_exec.js
```

而且，我们还有一个`net/http`基于简单的文件服务器，从[这里](https://github.com/golang/go/wiki/WebAssembly)再次被盗，以提供我们`index.html`和我们的各种其他WebAssembly文件：

```go
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

当您`localhost:8080`启动此服务器后导航到该服务器时，您应该看到该`Run`按钮是可单击的，如果您在浏览器中打开控制台，则`Hello World`每次单击按钮时都会看到它打印出来！

太棒了，我们成功地编译了一个非常简单的Go - > WebAssembly项目并让它在浏览器中运行。

# 一个更复杂的例子

现在为好一点。比如，我们想要创建一个更复杂的示例，其中包含DOM操作，可以绑定到按钮点击的自定义Go函数等等。谢天谢地，这并不太难！

## 注册功能

我们首先创建一些我们想要暴露给前端的函数。我今天的心情而喧宾夺主所以这些都将是公正`add`和`subtract`。

这些函数采用类型数组，`js.Value`并使用`js.Global().Set()`函数设置`output`为等于函数内完成的任何计算的结果。为了更好地衡量，我们还会向控制台打印出结果：

```go
func add(i []js.Value) {
	js.Global().Set("output", js.ValueOf(i[0].Int()+i[1].Int()))
	println(js.ValueOf(i[0].Int() + i[1].Int()).String())
}

func subtract(i []js.Value) {
	js.Global().Set("output", js.ValueOf(i[0].Int()-i[1].Int()))
	println(js.ValueOf(i[0].Int() - i[1].Int()).String())
}

func registerCallbacks() {
	js.Global().Set("add", js.NewCallback(add))
	js.Global().Set("subtract", js.NewCallback(subtract))
}

func main() {
	c := make(chan struct{}, 0)

	println("WASM Go Initialized")
	// register functions
	registerCallbacks()
	<-c
}
```

您会注意到我们`main`通过调用`make`和创建新频道略微修改了我们的功能。这有效地将我们以前短命的程序变成了一个长期运行的程序。我们还调用另一个`registerCallbacks()`几乎像路由器一样的函数，而是创建新的回调函数，有效地将我们新创建的函数绑定到我们的前端。

为了实现这一点，我们必须稍微修改我们的JavaScript代码，`index.html`以便在获取它时立即运行我们的程序实例：

```js
const go = new Go();
let mod, inst;
WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then(async (result) => {
	mod = result.module;
	inst = result.instance;
	await go.run(inst)
});
```

再次在浏览器中加载它，您应该看到，在没有任何按钮按下的情况下，`WASM Go Initialized`在控制台中打印出来。这意味着一切都有效。

然后我们可以从`<button>`像这样的元素开始调用这些函数：

```html
<button onClick="add(2,3);" id="addButton">Add</button>
<button onClick="subtract(10,3);" id="subtractButton">Subtract</button>
```

删除现有`Run`按钮并将这两个新按钮添加到您的`index.html`。当您在浏览器中重新加载页面并打开控制台时，您应该能够看到此功能的输出打印出来。

我们慢慢地，但肯定开始到这个地方！

## 评估DOM元素

因此，我想下一阶段是开始评估DOM元素并使用它们的值而不是硬编码值。

让我们修改`add()`函数，这样我就可以传入`id`2s的`<input/>`元素，然后像这样添加这些元素的值：

```go
func add(i []js.Value) {
	value1 := js.Global().Get("document").Call("getElementById", i[0].String()).Get("value").String()
	value2 := js.Global().Get("document").Call("getElementById", i[1].String()).Get("value").String()
	js.Global().Set("output", value1+value2)
	println(value1 + value2)
}
```

然后我们可以更新我们`index.html`的代码：

```html
<input type="text" id="value1"/>
<input type="text" id="value2"/>

<button onClick="add('value1', 'value2');" id="addButton">Add</button>
```

如果在我们的输入中输入一些数值，然后单击`Add`按钮，您应该会看到在控制台中打印出的两个值的串联。

我们忘记了什么？我们需要将这些字符串值解析为int值：

```go
func add(i []js.Value) {
	value1 := js.Global().Get("document").Call("getElementById", i[0].String()).Get("value").String()
	value2 := js.Global().Get("document").Call("getElementById", i[1].String()).Get("value").String()

	int1, _ := strconv.Atoi(value1)
	int2, _ := strconv.Atoi(value2)

	js.Global().Set("output", int1+int2)
	println(int1 + int2)
}
```

你可能会注意到我在这里没有处理错误，因为我感觉很懒，这只是为了表演。

尝试重新编译此代码并重新加载浏览器，您应该注意到，如果我们输入值`22`并`3`在我们的输入中，它会`25`在控制台中成功输出。

# 操纵DOM元素

如果我们的计算器实际上没有在我们的页面中报告结果，那么我们的计算器就不会很好，所以让我们现在解决这个问题`id`，我们将把结果输出到第三个：

```go
func add(i []js.Value) {
	value1 := js.Global().Get("document").Call("getElementById", i[0].String()).Get("value").String()
	value2 := js.Global().Get("document").Call("getElementById", i[1].String()).Get("value").String()

	int1, _ := strconv.Atoi(value1)
	int2, _ := strconv.Atoi(value2)

	js.Global().Get("document").Call("getElementById", i[2].String()).Set("value", int1+int2)
}
```

# 更新我们的减法函数：

最后，让我们更新我们的减法方法：

```go
func subtract(i []js.Value) {
	value1 := js.Global().Get("document").Call("getElementById", i[0].String()).Get("value").String()
	value2 := js.Global().Get("document").Call("getElementById", i[1].String()).Get("value").String()

	int1, _ := strconv.Atoi(value1)
	int2, _ := strconv.Atoi(value2)

	js.Global().Get("document").Call("getElementById", i[2].String()).Set("value", int1-int2)
}
```

我们完成的`index.html`应该看起来像这样：

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
</head>

<body>

	<script src="wasm_exec.js"></script>

	<script>
		if (!WebAssembly.instantiateStreaming) { // polyfill
			WebAssembly.instantiateStreaming = async (resp, importObject) => {
				const source = await (await resp).arrayBuffer();
				return await WebAssembly.instantiate(source, importObject);
			};
		}

		const go = new Go();
		let mod, inst;
		WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then(async (result) => {
			mod = result.module;
			inst = result.instance;
			await go.run(inst)
		});

	</script>

	<input type="text" id="value1"/>
	<input type="text" id="value2"/>

	<button onClick="add('value1', 'value2', 'result');" id="addButton">Add</button>
	<button onClick="subtract('value1', 'value2', 'result');" id="subtractButton">Subtract</button>

	<input type="text" id="result">

</body>

</html>
```

# 结论

因此，在本教程中，我们学习了如何使用Go语言的新v1.11将Go程序编译为WebAssembly。我们创建了一个非常简单的计算器，它将Go代码中的函数暴露给我们的前端，并且还进行了一些DOM解析和操作来启动。

希望你发现这篇文章有用/有趣！如果你这样做，那么我很乐意在下面的评论部分听到你的意见。如果您希望支持我的工作，请随时订阅我的YouTube频道：[TutorialEdge](https://youtube.com/tutorialedge)。