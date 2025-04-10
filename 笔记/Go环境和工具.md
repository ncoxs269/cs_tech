2025-03-24 21:55
Status: #idea
Tags: [[Go]]

# 1 编译
```bash
$ go build helloworld.go
```
这个命令生成一个名为 `helloworld` 的可执行的二进制文件（译注：Windows 系统下生成的可执行文件是 `helloworld.exe`，增加了 `.exe` 后缀名），之后你可以随时运行它（译注：在 Windows 系统下在命令行直接输入 `helloworld.exe` 命令运行），不需任何处理（译注：因为静态编译，所以不用担心在系统库更新的时候冲突，幸福感满满）。

# 2 环境
## 2.1 GOPATH
`GOPATH`是Go语言使用的一个环境变量，使用绝对路径提供项目的工作目录，适合处理大量 Go语言源码、多个包组合而成的复杂工程。实际使用中，可以先通过命令`go env`来查看一下当前的`GOPATH`值，然后再决定是不是需要重新设置。可以使用 `go env -w 环境变量=值` 来修改 `GOPATH`。
在 Go 1.8 版本之前，`GOPATH` 环境变量默认是空的。从 Go 1.8 版本开始，Go 开发包在安装完成后，将 `GOPATH` 赋予了一个默认的目录，参见下表。

| 平 台        | GOPATH 默认值       | 举 例             |
| ---------- | ---------------- | --------------- |
| Windows 平台 | %USERPROFILE%/go | C:\Users\用户名\go |
| Unix 平台    | $HOME/go         | /home/用户名/go    |
在 `GOPATH` 指定的工作目录下，代码总是会保存在 `GOPATH/src` 目录下。在工程经过 `go build`、`go install` 或 `go get` 等指令后，会将产生的二进制可执行文件放在 `GOPATH/bin` 目录下，生成的中间缓存文件会被保存在 `GOPATH/pkg` 下。
建议大家无论是使用命令行或者使用集成开发环境编译 Go 源码时，`GOPATH` 跟随项目设定。在 Jetbrains 公司的 GoLand 集成开发环境（IDE）中的 `GOPATH` 设置分为全局 `GOPATH` 和项目 `GOPATH`，项目`GOPATH`不会被设置到环境变量的 `GOPATH` 中，但会在编译时使用到这个目录。建议在开发时只填写项目 `GOPATH`，每一个项目尽量只设置一个 `GOPATH`，不使用多个 `GOPATH` 和全局的 `GOPATH`。

## 2.2 Go Module
最早的时候，Go语言所依赖的所有的第三方库都放在 GOPATH 这个目录下面，这就导致了同一个库只能保存一个版本的代码。如果不同的项目依赖同一个第三方的库的不同版本，应该怎么解决？
Go Module 是Go语言从 1.11 版本之后官方推出的版本管理工具，并且从 Go1.13 版本开始，Go Module 成为了Go语言默认的依赖管理工具。要使用 Go Module，首先需要把 golang 升级到 1.11 版本以上，然后设置 `GO111MODULE` 环境变量。

`GO111MODULE` 有如下值：
- `GO111MODULE=off` 禁用 go module，编译时会从 `GOPATH` 和 vendor 文件夹中查找包；
- `GO111MODULE=on` 启用 go module，编译时会忽略 `GOPATH` 和 vendor 文件夹，只根据 go.mod下载依赖；
- `GO111MODULE=auto`（默认值），当项目在 GOPATH/src 目录之外，并且项目根目录有 go.mod 文件时，开启 Go Module。
在开启 `GO111MODULE` 之后就可以使用 Go Module 工具了，也就是说在以后的开发中就没有必要在 `GOPATH` 中创建项目了，并且还能够很好的管理项目依赖的第三方包信息。

## 2.3 GOPROXY
proxy 顾名思义就是代理服务器的意思。大家都知道，国内的网络有防火墙的存在，这导致有些Go语言的第三方包我们无法直接通过`go get`命令获取。
`GOPROXY` 是Go语言官方提供的一种通过中间代理商来为用户提供包下载服务的方式。要使用 `GOPROXY` 只需要设置环境变量 `GOPROXY` 即可。
目前公开的代理服务器的地址有：
- goproxy.io；
- goproxy.cn：（推荐）由国内的七牛云提供。
    
我们可以用以下命令设置 `GOPROXY`：
```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

# 3 Go 命令
## 3.1 go mod 命令
常用的`go mod`命令如下表所示：

| 命令              | 作用                              |
| --------------- | ------------------------------- |
| go mod init     | 初始化当前文件夹，创建 go.mod 文件           |
| go mod download | 下载依赖包到本地（默认为 GOPATH/pkg/mod 目录） |
| go mod tidy     | 增加缺少的包，删除无用的包                   |
| go mod why      | 解释为什么需要依赖                       |
| go mod edit     | 编辑 go.mod 文件                    |
| go mod graph    | 打印模块依赖图                         |
| go mod vendor   | 将依赖复制到 vendor 目录下               |
| go mod verify   | 校验依赖                            |

## 3.2 使用go get命令下载指定版本的依赖包
执行`go get`命令，在下载依赖包的同时还可以指定依赖包的版本。
- 运行`go get -u`命令会将项目中的包升级到最新的次要版本或者修订版本；
- 运行`go get -u=patch`命令会将项目中的包升级到最新的修订版本；
- 运行`go get [包名]@[版本号]`命令会下载对应包的指定版本或者将对应包升级到指定的版本。

`go get [包名]@[版本号]` 命令中版本号可以是 x.y.z 的形式，例如 `go get foo@v1.2.3`，也可以是 git 上的分支或 tag，例如 `go get foo@master`，还可以是 git 提交时的哈希值，例如 `go get foo@e3702bed2`。

## 3.3 使用go doc查看Go包的文档
例如：
```bash
$ go doc text/template
$ go doc html/template
```

# 4 例子：初始化项目
在 `GOPATH` 目录之外新建一个目录，并使用`go mod init`初始化生成 go.mod 文件。
```bash
go mod init hello
go: creating new go.mod: module hello
```
初始化生成的 go.mod 文件如下所示：
```Go
module hello

go 1.13
```

新建一个 main.go 文件，写入以下代码：
```Go
package main

import (
    "net/http"
    "github.com/labstack/echo"
)

func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })
    e.Logger.Fatal(e.Start(":1323"))
}
```
执行`go run main.go`运行代码会发现 go mod 会自动查找依赖自动下载：
```bash
go run main.go
go: finding github.com/labstack/echo v3.3.10+incompatible
go: downloading github.com/labstack/echo v3.3.10+incompatible
go: extracting github.com/labstack/echo v3.3.10+incompatible
go: finding github.com/labstack/gommon v0.3.0
......
```
现在查看 go.mod 内容：
```Go
module hello

go 1.13

require (
    github.com/labstack/echo v3.3.10+incompatible // indirect
    github.com/labstack/gommon v0.3.0 // indirect
    golang.org/x/crypto v0.0.0-20191206172530-e9b2fee46413 // indirect
)
```

go module 安装 package 的原则是先拉取最新的 release tag，若无 tag 则拉取最新的 commit。go 会自动生成一个 go.sum 文件来记录 dependency tree：
```Go
github.com/davecgh/go-spew v1.1.0/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/labstack/echo v3.3.10+incompatible h1:pGRcYk231ExFAyoAjAfD85kQzRJCRI8bbnE7CX5OEgg=
github.com/labstack/echo v3.3.10+incompatible/go.mod h1:0INS7j/VjnFxD4E2wkz67b8cVwCLbBmJyDaka6Cmk1s=
github.com/labstack/gommon v0.3.0 h1:JEeO0bvc78PKdyHxloTKiF8BD5iGrH8T6MSeGvSgob0=
github.com/labstack/gommon v0.3.0/go.mod h1:MULnywXg0yavhxWKc+lOruYdAhDwPK9wf0OL7NoOu+k=
github.com/mattn/go-colorable v0.1.2 h1:/bC9yWikZXAL9uJdulbSfyVNIR3n3trXl+v8+1sx8mU=
... 
```

可以使用命令`go list -m -u all`来检查可以升级的 package，使用`go get -u need-upgrade-package`升级后会将新的依赖版本更新到 go.mod，也可以使用`go get -u`升级所有依赖。

# 竞争条件检测
Go的runtime和工具链为我们装备了一个复杂但好用的动态分析工具，竞争检查器（the race detector）。
只要在go build，go run或者go test命令后面加上-race的flag，就会使编译器创建一个你的应用的“修改”版或者一个附带了能够记录所有运行期对共享变量访问工具的test，并且会记录下每一个读或者写共享变量的goroutine的身份信息。另外，修改版的程序会记录下所有的同步事件，比如go语句，channel操作，以及对`(*sync.Mutex).Lock`，`(*sync.WaitGroup).Wait`等等的调用。
竞争检查器会检查这些事件，会寻找在哪一个goroutine中出现了这样的case，例如其读或者写了一个共享变量，这个共享变量是被另一个goroutine在没有进行干预同步操作便直接写入的。这种情况也就表明了是对一个共享变量的并发访问，即数据竞争。这个工具会打印一份报告，内容包含变量身份，读取和写入的goroutine中活跃的函数的调用栈。这些信息在定位问题时通常很有用。
竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你的包。

---
# 5 引用
https://gopl-zh.github.io/index.html