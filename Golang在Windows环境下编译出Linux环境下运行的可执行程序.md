# Golang在Windows环境下编译出Linux环境下运行的可执行程序

首先，获取目标系统所支持的构架，比如x86/x64/arm64/arm 等等。

在 linux 系统中，可以使用命令`uname -a`查看系统的一些信息；以作者的机器来说，如果你看到arm64之类的字样，表明你的系统是支持这种架构的程序的。

我们只需要将写好的go语言编译成这类架构的Linux程序即可。

在windows 系统CMD下，输入命令：`go env`查看 go的环境变量。

其中有两个参数对于跨平台编译至关重要。一个为`%GOOS`，一个为`%GOARCH`。在Windows系统下，%GOOS默认为 windows，%GOARCH 为 amd64 （根据系统不同，可能有所不同）。

要将程序编译为Linux程序，需设置 %GOOS 为 linux，且%GOARCH为Linux系统支持的架构。

在CMD下，输入：

```shell
go env -w GOOS=linux
go env -w GOARCH=amd64
```

Shell

然后编译GO程序即可，如无意外，会在工作目录生成一个无扩展名的文件，我们就可以在Linux系统下运行它了。

Linux系统下运行程序需要为新文件赋予可执行权限。

我们可以使用这种方法在Windows系统下编译出其它平台的可执行程序，相反，也可以在Linux系统下编译出Windows平台的可执行程序。

原创内容，如需转载，请注明出处；

本文地址： https://www.perfcode.com/p/1541.html