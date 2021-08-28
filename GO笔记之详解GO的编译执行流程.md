
# GO笔记之详解GO的编译执行流程 #

上篇文章介绍了Golang在不同系统下的安装，并完成了经典的Hello World案例。在这个过程中，我们用到了go run命令，它完成源码从编译到执行的整个过程。

![](./images/v2-ff20f94589120bd8eaee2b236060268a_720w.jpg)

今天来详细介绍下这个过程。简单理解，go run 可等价于 go build + 执行。

## build命令简述 ##

在Golang中，build过程主要由go build执行。它完成了源码的编译与可执行文件的生成。

go build接收参数为.go文件或目录，默认情况下编译当前目录下所有.go文件。在main包下执行会生成相应的可执行文件，在非main包下，它会做一些检查，生成的库文件放在缓存目录下，在工作目录下并无新文件生成。

## 新建hello案例 ##

在正式介绍编译流程前，再重新演示下Hello World案例，新建hello.go文件，代码如下：

```
package main

import "fmt"

func main() {
	fmt.Println("Hello World")
}
```

执行go build hello.go，目录下生成可执行文件hello。执行hello，输出Hello World。

## 介绍build选项 ##

编译流程的演示需要go build提供的几个选项协助，执行go help build查看。如下：

```
$ go help build

...

-n 不执行地打印流程中用到的命令
-x 执行并打印流程中用到的命令，要注意下它与-n选项的区别
-work 打印编译时的临时目录路径，并在结束时保留。默认情况下，编译结束会删除该临时目录。

...
```

这几个选项也适用于go run命令。有没有觉得和sh命令选项类似，可见计算机里的很多知识都是相通的。

## 打印执行流程 ##

使用 -n 选项在命令不执行的情况下，查看go build的执行流程，如下：

```
$ go build -n hello.go
#
# command-line-arguments
#

mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile fmt=/usr/local/go/pkg/darwin_amd64/fmt.a
packagefile runtime=/usr/local/go/pkg/darwin_amd64/runtime.a
EOF
cd /Users/polo/Public/Work/go/src/study/basic/hello
/usr/local/go/pkg/tool/darwin_amd64/compile -o $WORK/b001/_pkg_.a -trimpath $WORK/b001 -p main -complete -buildid fVbBEz0nTJc3r6VxU5ye/fVbBEz0nTJc3r6VxU5ye -goversion go1.11.1 -D _/Users/polo/Public/Work/go/src/study/basic/hello -importcfg $WORK/b001/importcfg -pack -c=4 ./hello.go
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a

...

packagefile internal/race=/usr/local/go/pkg/darwin_amd64/internal/race.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=P1Y_fbNXAEG6zEEGqFsM/fVbBEz0nTJc3r6VxU5ye/fVbBEz0nTJc3r6VxU5ye/P1Y_fbNXAEG6zEEGqFsM -extld=clang $WORK/b001/_pkg_.a
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out hello
```

过程看起来很乱，仔细观看下来可以发现主要由几部分组成，分别是：

- 创建临时目录，mkdir -p $WORK/b001/
- 查找依赖信息，cat >$WORK/b001/importcfg << ...
- 执行源代码编译，/usr/local/go/pkg/tool/darwin_amd64/compile ...
- 收集链接库文件，cat >$WORK/b001/importcfg.link << ...
- 生成可执行文件，/usr/local/go/pkg/tool/darwin_amd64/link -o ...
- 移动可执行文件，mv $WORK/b001/exe/a.out hello

如此一解释，build 的流程就很清晰了。如果是熟悉c/c++开发的朋友，会发现这个过程似曾相识。当然，相比之下c/c++还会多出一步预处理。

再来优化下之前的流程图，如下：

![](./images/v2-f346d6e11bb18f498c110110c900a04a_720w.jpg)

我们把build过程细化成两部分，compile与link，即编译和链接。此处用到了两个很重要的命令，complie和link。它们都是属于go tool的子命令。

## 说说run的流程 ##

理解了build过程，run就很好理解了。我们使用go run -x hello.go 查看执行过程，如下：

```
...
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/hello -importcfg $WORK/b001/importcfg.link -s -w -buildmode=exe -buildid=fveq2guPMmsyv8t4cV_M/xYBkVZeN1BHy2ygmstrB/pWJerx2-jOU98BpvIFO6/fveq2guPMmsyv8t4cV_M -extld=clang $WORK/b001/_pkg_.a
$WORK/b001/exe/hello
Hello World
```

重点看结尾部分，与build不同的是，在link生成hello文件后，并没有把它移动到当前目录，而是通过$WORK/b001/exe/hello执行了程序。加上编译，画出如下流程图：

![](./images/v2-201c0cc3abbb79d0b860b84f5f947c04_720w.jpg)

到此，run的整个流程到此就很清晰了。

## 通过--work保留可执行文件 ##

那么能否拿到这个临时生成的可执行文件？默认是不行的，在go run最后会把临时目录删除。我们可以使用--work保留这个目录。演示过程如下：

```
$ go run -x --work hello.go
WORK=/var/folders/bw/8yw8h4yj2vb6mxtb6t8t41f00000gn/T/go-build149627400

...

$WORK/b001/exe/hello
Hello World
```

打印了临时目录路径WORK，通过mv命令我们就可以把run生成的hello文件拷贝到当前目录，如下所示：

    $ mv /var/folders/bw/8yw8h4yj2vb6mxtb6t8t41f00000gn/T/go-build149627400b001/exe/hello hello

可以执行下hello看看和我们预期的是否一样。

## 总结 ##

本篇文章从go run引出Golang的编译执行流程。利用build提供的几个调试选项，我们实现了过程的逐步分解，最终比较详细地介绍了整个编译执行流程中的各个阶段。
