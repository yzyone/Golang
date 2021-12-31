# CGO 静态编译c相关组件 #

转载：https://www.iblo.info/posts/1004.html

CGO 静态编译c相关组件

起先我是直接在go文件里指定LDFLAG：

    //#cgo LDFLAGS: -static

编译的时候出现了如下错误：

    cannot load imported symbols from ELF file $WORK/_/home/goplayground/_obj/_cgo_.o: no symbol section

我还真是第一次碰到没有符号表的ELF。

google groups给予了详细解答

静态标志需要在编译go文件时由命令行参数指定：

    go build -ldflags -extldflags=-static

> On Thu, Oct 10, 2013 at 9:07 PM, James Bardin <j.ba…@gmail.com> wrote: I was trying to put together an example of statically linking a cgo binary, and I noticed that I can’t use -static in CGO_LDFLAGS. go build -ldflags -extldflags=-static works as expected, but if I need to provide other linking information I still need to put that in CGO_LDFLAGS. I was trying to get all the options into the source file so go build would “Just Work”.

> If LDFLAGS contains -static, I get: cannot load imported symbols from ELF file $WORK/_/home/jbardin/try/_obj/cgo.o: no symbol section We use -Wl,-r to link cgo intermediate object files, I guess -static is interfering with that.

> Looking over the -x output, when -static is in LDFLAGS the compilation step for cgo.o contains -static, whereas it does not when only using -extldflags right. CGO_LDFLAGS is meant for both internal linking and external linking mode, whereas -extldflags is meant only for external linking.

> Is this a bug, or just something that can’t be handled automatically? I don’t think it’s a bug. Static linking is still an area that needs work. For now, you can embed the static library as a .syso file (use ar x to extract its content, and use ld -r to link them into a single file, then rename it to *.syso, put it into the package directory); or you could put the C/C++ source files in the package directory and let the Go tool build the library for you.

> I prefer the 2nd solution as it should always work and don’t involve checking in binary blobs.

go做的真的绝。

转载于:https://my.oschina.net/chuqq/blog/1809679

相关资源：go语言写的静态网站_go开发静态网页-Web开发代码类资源