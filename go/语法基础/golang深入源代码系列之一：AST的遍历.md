# golang深入源代码系列之一：AST的遍历

# 怎么分析golang源代码

我们拿到一个golang的工程后（通常是个微服务），怎么从词法、语法的角度来分析源代码呢？golang提供了一系列的工具供我们使用：

- go/scanner包提供词法分析功能，将源代码转换为一系列的token，以供go/parser使用
- go/parser包提供语法分析功能，将这些token转换为AST（Abstract Syntax Tree, 抽象语法树）

## Scanner

- 任何编译器所做的第一步都是将源代码转换成token，这就是Scanner所做的事
- token可以是关键字，字符串值，变量名以及函数名等等
- 在golang中，每个token都以它所处的位置，类型和原始字面量来表示

比如我们用如下代码扫描源代码的token：

```
func TestScanner(t *testing.T) {
	src := []byte(`package main
import "fmt"
//comment
func main() {
  fmt.Println("Hello, world!")
}
`)

	var s scanner.Scanner
	fset := token.NewFileSet()
	file := fset.AddFile("", fset.Base(), len(src))
	s.Init(file, src, nil, 0)

	for {
		pos, tok, lit := s.Scan()
		fmt.Printf("%-6s%-8s%q\n", fset.Position(pos), tok, lit)

		if tok == token.EOF {
			break
		}
	}
}
```

结果：

```
1:1   package "package"
1:9   IDENT   "main"
1:13  ;       "\n"
2:1   import  "import"
2:8   STRING  "\"fmt\""
2:13  ;       "\n"
4:1   func    "func"
4:6   IDENT   "main"
4:10  (       ""
4:11  )       ""
4:13  {       ""
5:3   IDENT   "fmt"
5:6   .       ""
5:7   IDENT   "Println"
5:14  (       ""
5:15  STRING  "\"Hello, world!\""
5:30  )       ""
5:31  ;       "\n"
6:1   }       ""
6:2   ;       "\n"
6:3   EOF     ""
```

**注意没有扫描出注释，需要的话要将s.Init的最后一个参数改为scanner.ScanComments。**

看下go/token/token.go的源代码可知，token就是一堆定义好的枚举类型，对于每种类型的字面值都有对应的token。

```
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package token defines constants representing the lexical tokens of the Go
// programming language and basic operations on tokens (printing, predicates).
//
package token

import "strconv"

// Token is the set of lexical tokens of the Go programming language.
type Token int

// The list of tokens.
const (
	// Special tokens
	ILLEGAL Token = iota
	EOF
	COMMENT

	literal_beg
	// Identifiers and basic type literals
	// (these tokens stand for classes of literals)
	IDENT  // main
	INT    // 12345
	FLOAT  // 123.45
	IMAG   // 123.45i
	CHAR   // 'a'
	STRING // "abc"
	literal_end
	        ...略...
)
```

## Parser

- 当源码被扫描成token之后，结果就被传递给了Parser
- 将token转换为抽象语法树（AST）
- 编译时的错误也是在这个时候报告的

什么是AST呢，这篇文章[何为语法树](http://huang-jerryc.com/2016/03/15/%E4%BD%95%E4%B8%BA%E8%AF%AD%E6%B3%95%E6%A0%91/)讲的很好。简单来说，AST（Abstract Syntax Tree）是使用树状结构表示源代码的语法结构，树的每一个节点就代表源代码中的一个结构。

来看如下的例子：

```
func TestParserAST(t *testing.T) {
	src := []byte(`/*comment0*/
package main
import "fmt"
//comment1
/*comment2*/
func main() {
  fmt.Println("Hello, world!")
}
`)

	// Create the AST by parsing src.
	fset := token.NewFileSet() // positions are relative to fset
	f, err := parser.ParseFile(fset, "", src, 0)
	if err != nil {
		panic(err)
	}

	// Print the AST.
	ast.Print(fset, f)
}
```

结果很长就不贴出来了，整个AST的树形结构可以用如下图表示：

![img](http://image99.renyit.com/image/2019-01-14-1.png)

**同样注意没有扫描出注释，需要的话要将parser.ParseFile的最后一个参数改为parser.ParseComments。**再对照如下ast.File的定义：

```
type File struct {
	Doc        *CommentGroup   // associated documentation; or nil
	Package    token.Pos       // position of "package" keyword
	Name       *Ident          // package name
	Decls      []Decl          // top-level declarations; or nil
	Scope      *Scope          // package scope (this file only)
	Imports    []*ImportSpec   // imports in this file
	Unresolved []*Ident        // unresolved identifiers in this file
	Comments   []*CommentGroup // list of all comments in the source file
}
```

可知上述例子中的`/*comment0*/`对照结构中的Doc，是整个go文件的描述。和`//comment1 `以及`/*comment2*/`不同，后两者是Decls中的结构。

## 遍历AST

golang提供了ast.Inspect方法供我们遍历整个AST树，比如如下例子遍历整个example/test1.go文件寻找所有return返回的地方：

```
func TestInspectAST(t *testing.T) {
	// Create the AST by parsing src.
	fset := token.NewFileSet() // positions are relative to fset
	f, err := parser.ParseFile(fset, "./example/test1.go", nil, parser.ParseComments)
	if err != nil {
		panic(err)
	}

	ast.Inspect(f, func(n ast.Node) bool {
		// Find Return Statements
		ret, ok := n.(*ast.ReturnStmt)
		if ok {
			fmt.Printf("return statement found on line %v:\n", fset.Position(ret.Pos()))
			printer.Fprint(os.Stdout, fset, ret)
			fmt.Printf("\n")
			return true
		}
		return true
	})
}
```

example/test1.go代码如下：

```
package main

import "fmt"
import "strings"

func test1() {
	hello := "Hello"
	world := "World"
	words := []string{hello, world}
	SayHello(words)
}

// SayHello says Hello
func SayHello(words []string) bool {
	fmt.Println(joinStrings(words))
	return true
}

// joinStrings joins strings
func joinStrings(words []string) string {
	return strings.Join(words, ", ")
}
```

结果为：

```
return statement found on line ./example/test1.go:16:2:
return true
return statement found on line ./example/test1.go:21:2:
return strings.Join(words, ", ")
```

还有另一种方法遍历AST，构造一个ast.Visitor接口：

```
type Visitor int

func (v Visitor) Visit(n ast.Node) ast.Visitor {
	if n == nil {
		return nil
	}
	fmt.Printf("%s%T\n", strings.Repeat("\t", int(v)), n)
	return v + 1
}

func TestASTWalk(t *testing.T) {
	// Create the AST by parsing src.
	fset := token.NewFileSet() // positions are relative to fset
	f, err := parser.ParseFile(fset, "", "package main; var a = 3", parser.ParseComments)
	if err != nil {
		panic(err)
	}
	var v Visitor
	ast.Walk(v, f)
}
```

旨在递归地打印出所有的token节点，输出：

```
*ast.File
	*ast.Ident
	*ast.GenDecl
		*ast.ValueSpec
			*ast.Ident
			*ast.BasicLit
```

**以上基础知识主要参考文章How a Go Program Compiles down to Machine Code（译文：Go 程序到机器码的编译之旅）**。下面来点干货。

# 怎么找到特定的代码块

其实翻一翻网上将这个golang的ast的文章也不少，但是大多停留在上文的阶段，没有实际指导开发运用。那么我们假设现在有一个任务，拿到了一个别人的项目（俗称接盘侠），现在需要找到源文件中的这些地方：特征是调用了`context.WithCancel`函数，并且入参为nil。比如example/test2.go文件里面，有十多种可能：

```
package main

import (
	"context"
	"fmt"
)

func test2(a string, b int) {
	context.WithCancel(nil) //000

	if _, err := context.WithCancel(nil); err != nil { //111
		context.WithCancel(nil) //222
	} else {
		context.WithCancel(nil) //333
	}

	_, _ = context.WithCancel(nil) //444

	go context.WithCancel(nil) //555

	go func() {
		context.WithCancel(nil) //666
	}()

	defer context.WithCancel(nil) //777

	defer func() {
		context.WithCancel(nil) //888
	}()

	data := map[string]interface{}{
		"x2": context.WithValue(nil, "k", "v"), //999
	}
	fmt.Println(data)

	/*
		for i := context.WithCancel(nil); i; i = false {//aaa
			context.WithCancel(nil)//bbb
		}
	*/

	var keys []string = []string{"ccc"}
	for _, k := range keys {
		fmt.Println(k)
		context.WithCancel(nil)
	}
}
```

从000到ccc，对应golang的AST的不同结构类型，现在需要把他们全部找出来。其中bbb这种情况代表了for语句，只不过在`context.WithCancel`函数不适用，所以注掉了。为了解决这个问题，首先需要仔细分析go/ast的Node接口。

## AST的结构定义

go/ast/ast.go中指明了ast节点的定义：

```
// All node types implement the Node interface.
type Node interface {
	Pos() token.Pos // position of first character belonging to the node
	End() token.Pos // position of first character immediately after the node
}

// All expression nodes implement the Expr interface.
type Expr interface {
	Node
	exprNode()
}

// All statement nodes implement the Stmt interface.
type Stmt interface {
	Node
	stmtNode()
}

// All declaration nodes implement the Decl interface.
type Decl interface {
	Node
	declNode()
}
```

语法有三个主体：表达式(expression)、语句(statement)、声明(declaration)，Node是基类，用于标记该节点的位置的开始和结束。而三个主体的函数没有实际意义，只是用三个interface来划分不同的语法单位,如果某个语法是Stmt的话,就实现一个空的stmtNode函数即可。参考这篇文章[go-parser-语法分析](https://studygolang.com/articles/6709)，定义了源文件中可能出现的语法结构。列表如下：

### 普通Node,不是特定语法结构,属于某个语法结构的一部分.

- Comment 表示一行注释 // 或者 / /
- CommentGroup 表示多行注释
- Field 表示结构体中的一个定义或者变量,或者函数签名当中的参数或者返回值
- FieldList 表示以”{}”或者”()”包围的Filed列表

### Expression & Types (都划分成Expr接口)

- BadExpr 用来表示错误表达式的占位符
- Ident 比如报名,函数名,变量名
- Ellipsis 省略号表达式,比如参数列表的最后一个可以写成arg…
- BasicLit 基本字面值,数字或者字符串
- FuncLit 函数定义
- CompositeLit 构造类型,比如{1,2,3,4}
- ParenExpr 括号表达式,被括号包裹的表达式
- SelectorExpr 选择结构,类似于a.b的结构
- IndexExpr 下标结构,类似这样的结构 expr[expr]
- SliceExpr 切片表达式,类似这样 expr[low:mid:high]
- TypeAssertExpr 类型断言类似于 X.(type)
- CallExpr 调用类型,类似于 expr()
- StarExpr 表达式,类似于 X
- UnaryExpr 一元表达式
- BinaryExpr 二元表达式
- KeyValueExp 键值表达式 key:value
- ArrayType 数组类型
- StructType 结构体类型
- FuncType 函数类型
- InterfaceType 接口类型
- MapType map类型
- ChanType 管道类型

### Statements

- BadStmt 错误的语句
- DeclStmt 在语句列表里的申明
- EmptyStmt 空语句
- LabeledStmt 标签语句类似于 indent:stmt
- ExprStmt 包含单独的表达式语句
- SendStmt chan发送语句
- IncDecStmt 自增或者自减语句
- AssignStmt 赋值语句
- GoStmt Go语句
- DeferStmt 延迟语句
- ReturnStmt return 语句
- BranchStmt 分支语句 例如break continue
- BlockStmt 块语句 {} 包裹
- IfStmt If 语句
- CaseClause case 语句
- SwitchStmt switch 语句
- TypeSwitchStmt 类型switch 语句 switch x:=y.(type)
- CommClause 发送或者接受的case语句,类似于 case x <-:
- SelectStmt select 语句
- ForStmt for 语句
- RangeStmt range 语句

### Declarations

- Spec type
- - Import Spec
- - Value Spec
- - Type Spec
- BadDecl 错误申明
- GenDecl 一般申明(和Spec相关,比如 import “a”,var a,type a)
- FuncDecl 函数申明

### Files and Packages

- File 代表一个源文件节点,包含了顶级元素.
- Package 代表一个包,包含了很多文件.

## 全类型匹配

那么我们需要仔细判断上面的总总结构，来适配我们的特征：

```
package go_code_analysis

import (
	"fmt"
	"go/ast"
	"go/token"
	"log"
)

var GFset *token.FileSet
var GFixedFunc map[string]Fixed //key的格式为Package.Func

func stmtCase(stmt ast.Stmt, todo func(call *ast.CallExpr) bool) bool {
	switch t := stmt.(type) {
	case *ast.ExprStmt:
		log.Printf("表达式语句%+v at line:%v", t, GFset.Position(t.Pos()))
		if call, ok := t.X.(*ast.CallExpr); ok {
			return todo(call)
		}
	case *ast.ReturnStmt:
		for i, p := range t.Results {
			log.Printf("return语句%d:%v at line:%v", i, p, GFset.Position(p.Pos()))
			if call, ok := p.(*ast.CallExpr); ok {
				return todo(call)
			}
		}
	case *ast.AssignStmt:
		//函数体里的构造类型 999
		for _, p := range t.Rhs {
			switch t := p.(type) {
			case *ast.CompositeLit:
				for i, p := range t.Elts {
					switch t := p.(type) {
					case *ast.KeyValueExpr:
						log.Printf("构造赋值语句%d:%+v at line:%v", i, t.Value, GFset.Position(p.Pos()))
						if call, ok := t.Value.(*ast.CallExpr); ok {
							return todo(call)
						}
					}
				}
			}
		}
	default:
		log.Printf("不匹配的类型:%T", stmt)
	}
	return false
}

//调用函数的N种情况
//对函数调用使用todo适配，并返回是否适配成功
func AllCallCase(n ast.Node, todo func(call *ast.CallExpr) bool) (find bool) {

	//函数体里的直接调用 000
	if fn, ok := n.(*ast.FuncDecl); ok {
		for i, p := range fn.Body.List {
			log.Printf("函数体表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
			find = find || stmtCase(p, todo)
		}

		log.Printf("func:%+v done", fn.Name.Name)
	}

	//if语句里
	if ifstmt, ok := n.(*ast.IfStmt); ok {
		log.Printf("if语句开始:%T %+v", ifstmt, GFset.Position(ifstmt.If))

		//if的赋值表达式 111
		if a, ok := ifstmt.Init.(*ast.AssignStmt); ok {
			for i, p := range a.Rhs {
				log.Printf("if语句赋值%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
				switch call := p.(type) {
				case *ast.CallExpr:
					c := todo(call)
					find = find || c
				}
			}
		}

		//if的花括号里面 222
		for i, p := range ifstmt.Body.List {
			log.Printf("if语句内部表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
			c := stmtCase(p, todo)
			find = find || c
		}

		//if的else里面 333
		if b, ok := ifstmt.Else.(*ast.BlockStmt); ok {
			for i, p := range b.List {
				log.Printf("if语句else表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
				c := stmtCase(p, todo)
				find = find || c
			}
		}

		log.Printf("if语句结束:%+v done", GFset.Position(ifstmt.End()))
	}

	//赋值语句 444
	if assign, ok := n.(*ast.AssignStmt); ok {
		log.Printf("赋值语句开始:%T %s", assign, GFset.Position(assign.Pos()))
		for i, p := range assign.Rhs {
			log.Printf("赋值表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
			switch t := p.(type) {
			case *ast.CallExpr:
				c := todo(t)
				find = find || c
			case *ast.CompositeLit:
				for i, p := range t.Elts {
					switch t := p.(type) {
					case *ast.KeyValueExpr:
						log.Printf("构造赋值%d:%+v at line:%v", i, t.Value, GFset.Position(p.Pos()))
						if call, ok := t.Value.(*ast.CallExpr); ok {
							c := todo(call)
							find = find || c
						}
					}
				}
			}
		}
	}

	if gostmt, ok := n.(*ast.GoStmt); ok {
		log.Printf("go语句开始:%T %s", gostmt.Call.Fun, GFset.Position(gostmt.Go))

		//go后面直接调用 555
		c := todo(gostmt.Call)
		find = find || c

		//go func里面的调用 666
		if g, ok := gostmt.Call.Fun.(*ast.FuncLit); ok {
			for i, p := range g.Body.List {
				log.Printf("go语句表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
				c := stmtCase(p, todo)
				find = find || c
			}
		}

		log.Printf("go语句结束:%+v done", GFset.Position(gostmt.Go))
	}

	if deferstmt, ok := n.(*ast.DeferStmt); ok {
		log.Printf("defer语句开始:%T %s", deferstmt.Call.Fun, GFset.Position(deferstmt.Defer))

		//defer后面直接调用 777
		c := todo(deferstmt.Call)
		find = find || c

		//defer func里面的调用 888
		if g, ok := deferstmt.Call.Fun.(*ast.FuncLit); ok {
			for i, p := range g.Body.List {
				log.Printf("defer语句内部表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
				c := stmtCase(p, todo)
				find = find || c
			}
		}

		log.Printf("defer语句结束:%+v done", GFset.Position(deferstmt.Defer))
	}

	if fostmt, ok := n.(*ast.ForStmt); ok {
		//for语句对应aaa和bbb
		log.Printf("for语句开始:%T %s", fostmt.Body, GFset.Position(fostmt.Pos()))
		for i, p := range fostmt.Body.List {
			log.Printf("for语句函数体表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
			c := stmtCase(p, todo)
			find = find || c
		}
	}

	if rangestmt, ok := n.(*ast.RangeStmt); ok {
		//range语句对应ccc
		log.Printf("range语句开始:%T %s", rangestmt.Body, GFset.Position(rangestmt.Pos()))
		for i, p := range rangestmt.Body.List {
			log.Printf("range语句函数体表达式%d:%T at line:%v", i, p, GFset.Position(p.Pos()))
			c := stmtCase(p, todo)
			find = find || c
		}
	}

	return
}

type FindContext struct {
	File      string
	Package   string
	LocalFunc *ast.FuncDecl
}

func (f *FindContext) Visit(n ast.Node) ast.Visitor {
	if n == nil {
		return f
	}

	if fn, ok := n.(*ast.FuncDecl); ok {
		log.Printf("函数[%s.%s]开始 at line:%v", f.Package, fn.Name.Name, GFset.Position(fn.Pos()))
		f.LocalFunc = fn
	} else {
		log.Printf("类型%T at line:%v", n, GFset.Position(n.Pos()))
	}

	find := AllCallCase(n, f.FindCallFunc)

	if find {
		name := fmt.Sprintf("%s.%s", f.Package, f.LocalFunc.Name)
		GFixedFunc[name] = Fixed{FuncDesc: FuncDesc{f.File, f.Package, f.LocalFunc.Name.Name}}
	}

	return f
}

func (f *FindContext) FindCallFunc(call *ast.CallExpr) bool {
	if call == nil {
		return false
	}

	log.Printf("call func:%+v, %v", call.Fun, call.Args)

	if callFunc, ok := call.Fun.(*ast.SelectorExpr); ok {
		if fmt.Sprint(callFunc.X) == "context" && fmt.Sprint(callFunc.Sel) == "WithCancel" {
			if len(call.Args) > 0 {
				if argu, ok := call.Args[0].(*ast.Ident); ok {
					log.Printf("argu type:%T, %s", argu.Name, argu.String())
					if argu.Name == "nil" {
						location := fmt.Sprint(GFset.Position(argu.NamePos))
						log.Printf("找到关键函数:%s.%s at line:%v", callFunc.X, callFunc.Sel, location)
						return true
					}
				}
			}
		}
	}

	return false
}
```

在`AllCallCase`方法中我们穷举了所有的调用函数的情况（ast.CallExpr），分别对应了000到ccc这13种情况。`stmtCase`方法分析了语句的各种可能，尽量找全所有。 `FindContext.FindCallFunc`方法首先看调用函数是不是选择结构，类似于a.b的结构；然后对比了调用函数的a.b是不是我们关心的`context.WithCancel`；最后看第一个实参的名称是不是`nil`。

最终找到了所有特征点：

```
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:9:21
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:11:34
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:12:22
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:14:22
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:11:34
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:17:28
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:19:24
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:22:22
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:25:27
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:28:22
2019/01/16 20:19:52 找到关键函数:context.WithCancel at line:./example/test2.go:45:22
```

故事的结尾，我们使用FindContext提供的walk方法递归了AST树，找到了所有符合我们特征的函数，当然例子里就`test`一个函数。所有代码都在<https://github.com/baixiaoustc/go_code_analysis>中能找到。