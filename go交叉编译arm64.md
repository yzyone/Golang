# go交叉编译arm64

golang一份代码可以编译出在不同系统和cpu架构运行的二进制文件。go也提供了很多环境变量，我们可以设置环境变量的值，来编译不同目标平台。

GOOS=linux GOARCH=$go_arch go build -ldflags "-s -w" mydaemon.go
1
指定叉编译目标：
GOARCH 目标架构（编译后的目标架构）的处理器架构（386、amd64、arm）
GOOS 目标平台（编译后的目标平台）的操作系统（darwin、freebsd、linux、windows）

（一）Windows 下编译Linux 64位可执行程序：

    SET CGO_ENABLED=0  //不设置也可以，原因不明
    SET GOOS=linux
    SET GOARCH=amd64
    通过 go env 查看设置是否成功。

（二）Linux 下编译Windows可执行程序：

    export CGO_ENABLED=0
    export GOOS=windows
    export GOARCH=amd64
    通过 go env 查看设置是否成功。
    go build hello.go

GOOS和GOARCH都有多个选项,可组合
————————————————
版权声明：本文为CSDN博主「西京刀客」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/inthat/article/details/120327137