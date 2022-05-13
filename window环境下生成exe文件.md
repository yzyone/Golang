# window环境下生成exe文件 #

windows系统使用的

	go build .

liunx系统使用：

    set GOARCH=amd64
    set GOOS=linux
    go build main.go

记得结束的时候： 
 
	SET  GOOS=windows

改变成之前的windows系统

// 编译适用于本机的版本

	go build
 
// 编译 Linux 版本

	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
 
// 编译 Windows 64 位版本

	CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
 
// 编译 MacOS 版本

	CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build