# GO环境安装配置
##go语言开发包安装
​    1、到 http://www.golangtc.com/ 下载最新的安装包
​    2、下载后直接双击msi文件安装，默认安装在c:\go
​    3、安装完成后默认会在环境变量 Path 后添加 Go 安装目录下的 bin 目录 C:\Go\bin\，并添加环境变量 GOROOT，值为 Go 安装根目录 C:\Go\
​    4、验证是否安装成功，在运行中输入 cmd 打开命令行工具，在提示符下输入 go
​    5、设置工作空间 gopath 目录(Go语言开发的项目路径)
​        Windows 设置如下，新建一个环境变量名称叫做GOPATH，值为你的工作目录，例如笔者的设置GOPATH=e:\mygo
​    6、用 go env 命令查看环境变量设置

### go语言eclipse开发环境搭建
​    1、点击help-> Install New Software… 菜单项
​    2、点击“Add…”按钮手动添加，在”Name“内输入：Goclipse Site，在”Location“内输入：http://goclipse.github.io/releases/，输入完成后，直接点击”OK“按钮即可
​    3、选中框内的 GoClipse 前的复选框，直接点击”Finish”按钮，直到安装完成。
​    4、打开 window -> preferences 设置 go 语言的安装目录和工作目录，以及插件



### go mod 翻墙

引用`golang.org/x/text`等官方包被墙时，设置下代理即可：

```bash
export GOPROXY=https://goproxy.io
```