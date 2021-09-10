
# Linux下安装Go环境 #

亲测可用，ubuntu18.04，转载自`https://www.jianshu.com/p/c43ebab25484`

## 安装Go环境 ##

Golang官网下载地址：`https://golang.org/dl/`

1、打开官网下载地址选择对应的系统版本, 复制下载链接
这里我选择的是

go1.11.5.linux-amd64.tar.gz：`https://dl.google.com/go/go1.11.5.linux-amd64.tar.gz`
 

image.png


2、cd进入你用来存放安装包的目录，我习惯在~下面创建个go文件夹。

    # 在 ~ 下创建 go 文件夹，并进入 go 文件夹
    mkdir ~/go && cd ~/go
    下载的 go 压缩包
    wget https://dl.google.com/go/go1.11.5.linux-amd64.tar.gz

3、下载完成


4、执行tar解压到/usr/loacl目录下（官方推荐），得到go文件夹等

    tar -C /usr/local -zxvf  go1.11.5.linux-amd64.tar.gz

5、添加/usr/loacl/go/bin目录到PATH变量中。添加到/etc/profile 或$HOME/.profile都可以

    # 习惯用vim，没有的话可以用命令`sudo apt-get install vim`安装一个
    vim /etc/profile
    # 在最后一行添加
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOROOT/bin
    # 保存退出后source一下（vim 的使用方法可以自己搜索一下）
    source /etc/profile

6、执行go version，如果现实版本号，则Go环境安装成功。是不是很简单呢？
 

## 运行第一个程序 ##

1、先创建你的工作空间(Workspaces)，官方建议目录$HOME/go。

    mkdir $HOME/go

2、将你的工作空间路径声明到环境变量中。和上一部分的第5步相似。

    # 编辑 ~/.bash_profile 文件
    vim ~/.bash_profile
    # 在最后一行添加下面这句。$HOME/go 为你工作空间的路径，你也可以换成你喜欢的路径
    export GOPATH=$HOME/go
    # 保存退出后source一下（vim 的使用方法可以自己搜索一下）
    source ~/.bash_profile

3、在你的工作空间创建你的第一个工程目录

    # 创建并进入你的第一个工程目录
    mkdir -p $GOPATH/src/hello && cd $GOPATH/src/hello

4、在你的工程目录下创建名为hello.go的文件

    vim hello.go

5、将下面内容粘贴到 hello.go 文件

```
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```

6、好了，工程目录和工程文件都准备好了。现在我们到我们的工程目录($GOPATH/src/hello)下构建我们的工程

    # 如果你当前的目录不在 $GOPATH/src/hello， 需要先执行 "cd $GOPATH/src/hello" 进入该目录
    # 执行构建工程的命令
    go build

7、等一会，命令执行完之后你可以看到目录下会多出一个 hello 的文件，这就是我们编译之后的文件啦。怎么执行我们的程序呢？只需要在当前目录下执行./xxx就可以啦！是不是敲鸡煎蛋呢！

    ./hello
 

## 关于Go的一些介绍 ##

**环境变量：**

- $GOROOT: 表示Go的安装目录。也就是上面我们解压出来的文件夹里面的go文件夹。
- $GOPATH: 表示我们的工作空间。用来存放我们的工程目录的地方。

**GOPATH目录：**

一般来说GOPATH下面会有三个文件夹：bin、pkg、src，没有的话自己创建。每个文件夹都有其的作用。

- bin：编译后可的执行文件的存放路径
- pkg：编译包时，生成的.a文件的存放路径
- src：源码路径，一般我们的工程就创建在src下面。

注意：如果要用Go Mod(Go1.11及以上支持)进行包管理，则需要在 GOPATH 以外的目录创建工程。关于Go Mod的使用，可以自行Google一下，这里就不赘述了。

原文链接：https://www.cnblogs.com/yiyi20120822/p/11652612.html