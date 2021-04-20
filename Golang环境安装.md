
# Golang环境安装 #

## ZERO ##

    持续更新 请关注：https://zorkelvll.cn/blogs/zorkelvll/articles/2018/11/02/1541171777964

## 一、Linux-CentOS7.2下安装 ##

    本文采用go的源码安装方式，安装golang1.11.1版本，其中由于go1.5版本以上在安装时会报ERROR: Cannot find /root/go1.4/bin/go 错误信息，这是因为go1.5版本以上编译安装时，需要go1.4版本，因此先安装好1.4版本的go：

1、安装go1.4

```
cd ~ && wget https://dl.google.com/go/go1.4.linux-amd64.tar.gz  #下载go1.4
tar -zxvf go1.4.linux-amd64.tar.gz   #解压go1.4
cd go/src/ && ./all.bash  #安装go1.4，若缺少gcc则先yum install gcc ;其中的test失败可以不关心之，改成./make.bash
cd ../.. && mv go /root/go1.4  #安装好的go项目移动至/root/go1.4
```

2、安装go1.11.1

```
cd ~/app &&  wget https://dl.google.com/go/go1.11.1.linux-amd64.tar.gz  #下载go1.11.1
tar -zxvf go1.11.1.linux-amd64.tar.gz   #解压go1.11.1
cd go/src/ && ./all.bash  #安装go1.11.1
```

若报错误：

```
go build bootstrap/cmd/compile/internal/ssa: /root/go1.4/pkg/tool/linux_amd64/6g: signal: killed

go tool dist: FAILED: /root/go1.4/bin/go install -gcflags=-l -tags=math_big_pure_go compiler_bootstrap bootstrap/cmd/...: exit status 1
```

则是因为系统内存不足，至少需要1G的内存类构建包...增加内存这里是

3、配置环境变量

```
vim /etc/profile   #添加以下配置
export GOROOT=/root/app/go1.11.1
export GOPATH=/root/project/gopath #其中gopath下建目录pkg,bin,src
export GOBIN=${GOPATH}/bin
export PATH=${PATH}:${GOBIN}:${GOROOT}/bin

#校验go环境
source /etc/profile
env | grep GO
echo $PATH
go version
```

## 二、Mac-macOS10.13.6下安装 ##

1、下载安装

    brew install go
    go version

2、配置环境变量

```
vim ~/.bash_profile
GOROOT=/usr/local/Cellar/go/1.11.4/libexec
export GOROOT
export GOPATH=/Users/zorke/project/gopath
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN:$GOROOT/bin
```

说明：

GOROOT： go安装目录

GOPATH：go工作目录,存在src\pkg\bin三个目录(可手动创建)

src目录: go的源文件

pkg目录: 编译好的库文件，主要是*.a文件;

bin目录: 可执行文件

GOBIN：go可执行文件目录

PATH：go可执行文件

查看配置`$ go env`

3、将各个go项目所在目录ln链接至gopath下

```
cd ~/project/zorke
ln -sv czk-blog/ ~/project/gopath/czk-blog
vim ~/.bash_profile
export GOPATH=/Users/zorke/project/gopath/czk-blog  
```

作者：zorkelvll

链接：https://www.jianshu.com/p/665b9cef79c1

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。