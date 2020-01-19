### 本文导航
1. [总览](#1)
2. [go build](#2)
3. [go install](#3)
4. [go get](#4)
5. [go clean](#5)
6. [go run](#6)
7. [go test](#7)
8. [go fmt](#8)
9. [go list](#9)
10. [go generate](#10)
11. [go tool pprof](#11)
12. [go mod](#12)

<h2 id="1">总览</h2>

`go` 命令是用于管理 go 源码的工具。

用法：  
```
go <command> [arguments]
```

其中 command 是 go 的子命令，有以下这些：

子命令 | 备注
---|---
bug | 直接在官方 GitHub 创建一个新的 issue
build | 编译源码包及其依赖
clean | 删除其他命令产生的一些文件以及缓存文件
doc | show documentation for package or symbol
env | 打印go的环境变量信息
fix | 将包中老版本代码更新为新版本的
fmt | 代码格式化
generate | 执行Go源码生成文件
get | 从远端代码库拉取并安装
install | 编译源码包及其依赖，并安装
list | 列出指定代码包或模块的信息
mod | 管理 module
run | 编译并运行可执行文件
test | 运行并测试包中的测试文件
tool | 运行特定的工具
version | 打印 go 版本
vet | 检查并报告包中源码的错误


<h2 id="2"> go build </h2>

#### 用法
```
go build [-o output] [-i] [build flags] [packages]
```

- 无参数：  
在要编译的包路径下执行 `go build`， 将在该路径下生成对应的可执行文件 `package.exe`

- 指定包：  
设置项目的根路径为 `GOPATH`，然后就可以在任意目录下执行 `go build xxx/abc`，其绝对路径为 `$GOPATH/src/xxx/abc`

- 指定文件列表：
`go build xxx/abc/a.go xxx/abc/b.go` 要求所有文件在同一个路径下

#### 注意事项
1. 路径中不要有中文
2. 包内或文件列表中不可有多个命令源码文件（即有 main 函数的文件），否则会报重复定义错误
3. 编译 main 包时，包内有且仅能有一个 main 函数
4. 在只编译库源码文件时，将只做检查，而不输出任何文件
5. 编译时将自动忽略以 “`_test.go`” 结尾的文件

#### 常见附加参数
附加参数 | 备注
--- | ---
-o | 指定输出可执行文件名
-v | 编译时显示包名
-i | 编译命令源码文件时，将安装其依赖的库源码文件到 GOPATH 的 pkg 目录下
-p n | 开启并发编译，默认情况下该值为 CPU 逻辑核数
-a | 强制重新构建，即使上次构建后无修改过
-n | 打印编译时会用到的所有命令，但不真正执行
-x | 打印编译时会用到的所有命令，且执行
-race | 开启竞态检测
-work | 打印出编译时生成的临时工作目录的路径，并在编译结束时保留它

<h2 id="3"> go install </h2>

go install 和 go build 相比，做的工作要多一步，即将归档文件（`.a`后缀的）放到 GOPATH 的 pkg 目录下，以及将可执行文件放到 bin 目录下。  
若有指定 GOBIN 的话，就将可执行文件放到 `$GOBIN` 中

go install 可以用 go build 的所有附件参数

<h2 id="4"> go get </h2>

go get 命令可以借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装。可以简单地理解为 “拉取代码 + go install”。  

这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad。在使用 go get 命令前，需要安装与远程包匹配的代码管理工具，如 Git、SVN、HG 等，参数中需要提供一个包名。

go get 可以用 go build 的所有附加参数，此外，还有一些独有的参数：

附加参数 | 备注
--- | ---
-d | 只下载，不安装
-f | 仅在使用-u标记时才有效。忽略掉对已下载代码包的导入路径的检查。如果下载并安装的代码包所属的项目是你从别人那里Fork过来的，那么这样做就尤为重要了。
-fix | 下载代码包后先执行修正动作，而后再进行编译和安装。
-insecure | 允许命令程序使用非安全的scheme（如HTTP）去下载指定的代码包。如果你用的代码仓库（如公司内部的Gitlab）没有HTTPS支持，可以添加此标记。
-t | 同时下载并安装指定的代码包中的测试源码文件中依赖的代码包。
-u | 更新已有代码包及其依赖包。默认情况下，该命令只会从网络上下载本地不存在的代码包，而不会更新已有的代码包。

<h2 id="5"> go clean </h2>

执行 go clean 命令会删除掉执行其它命令时产生的一些文件和目录，包括：
1. 使用 go build 命令时在当前代码包下生成的与包名同名或者与Go源码文件同名的可执行文件；
2. 执行 go test 命令并加入-c标记时在当前代码包下生成的以包名加“.test”后缀为名的文件。在Windows下，则是以包名加“.test.exe”后缀为名的文件；
3. 执行 go clean -i 将删除 go install 安装的文件；
4. 还有一些目录和文件是在编译Go或C源码文件时留在相应目录中的。包括：“_obj”和“_test”目录，名称为“_testmain.go”、“test.out”、“build.out”或“a.out”的文件，名称以“.5”、“.6”、“.8”、“.a”、“.o”或“.so”为后缀的文件。这些目录和文件是在执行go build命令时生成在临时目录中的。临时目录的名称以go-build为前缀。
5. 执行 go clean -r 将删除包括当前代码包的所有依赖包的上述目录和文件。

<h2 id="6"> go run </h2>

go run 将编译并执行可执行文件，但是不生成文件。（实际是在临时文件夹中生成了，运行完又删除了）
```
go run test.go [arg1 ...]
```
其中 arg 可作为代码可以接收的命令行输入提供给程序。

<h2 id="7"> go test </h2>

go test 用于对Go语言编写的程序进行测试。  

go test 命令中包含了编译动作，所以它可以接受可用于go build命令的所有附加参数。  

go test -c 可以生成名为 pkg.test 的可执行文件，但不执行。

<h2 id="8"> go fmt 和 gofmt </h2>

两者都用于对Go源码按代码规范进行格式化。`go fmt xxx.go` 等同于 `gofmt -l -w xxx.go`。

<h2 id="9"> go list </h2>

go list 命令的作用是列出指定的代码包的信息。默认只打印导入路径。

- -json 以JSON格式输出
- -f {{.XX}} 只看XX属性

<h2 id="10"> go generate </h2>

`go generate` 命令是在Go语言 1.4 版本里面新添加的一个命令，当运行该命令时，它将扫描与当前包相关的源代码文件，找出所有包含`//go:generate`的特殊注释，提取并执行该特殊注释后面的命令。

如下代码：
```go
package main
//go:generate go run main.go
//go:generate go version
func main() {
	// ...
}
```
在命令行执行 `go generate`，则会依次执行注释中的两行命令。

[应用示例](https://segmentfault.com/a/1190000020158429)

使用go generate命令时有以下几点需要注意：
1. 该特殊注释必须在 .go 源码文件中；
2. 每个源码文件可以包含多个 generate 特殊注释；
3. 运行go generate命令时，才会执行特殊注释后面的命令；
4. 当go generate命令执行出错时，将终止程序的运行；
5. 特殊注释必须以//go:generate开头，双斜线后面没有空格。

<h2 id="11"> go tool pprof </h2>

Go语言工具链中的 `go tool pprof` 可以帮助开发者快速分析及定位各种性能问题，如 CPU 消耗、内存分配及阻塞分析。

[点击查阅](https://hyper0x.github.io/go_command_tutorial/#/0.12)

<h2 id="12"> go mod </h2>

[go mod 使用](https://segmentfault.com/a/1190000018536993)
