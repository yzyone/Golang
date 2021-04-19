# go编译安装 #

设置环境变量

    # sudo vim /etc/profile.d/go
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOROOT/bin
    export GOPATH=/root/Go
    # source /etc/profile.d/go

安装C工具

	# sudo apt-get install bison ed gawk gcc libc6-dev make

安装1.4: 如果需要安装1.4以上的版本，需要把1.4版本一起下载。

    # tar -zxvf go1.4.linux-amd64.tar.gz
    # mv go  /home/evescn/go1.4

安装1.4编译器

    # cd go1.4/src/
    # sudo CGO_ENABLED=0
    # ./make.bash

安装1.10.1

    # tar -zxvf go1.10.linux-amd64.tar.gz
    # mv go /home/evescn
    # cd /home/evescn/go/src
    # ./all.bash

测试代码：

```
# vim hello.go
package main
 
        func main() {
            println("Hello", "world")
        }
# go run hello.go
```

go编译go代码：

编译

	# go build test.go
	# 输入可执行文件test
	# ./test 运行go代码

指定输出文件

	# go build -o evescn test.go

修改权限命令

    # chmod 777 程序名称

后台运行的命令

	# nohup ./程序名 & 

不输出错误信息

	# nohup ./程序名 >/dev/null 2>&1 &

关闭程序

    # ps aux | grep '程序名'
    # kill '进程ID'

转载于:https://www.cnblogs.com/python-gm/p/10692317.html

相关资源：[GO编译开发环境安装包-linux官方版](https://download.csdn.net/download/songylwq/10778034?spm=1001.2101.3001.5697)