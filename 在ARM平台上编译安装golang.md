
# 在ARM平台上编译安装golang #

golang也就是go语言，现在已经发行到1.4.1版本了，语言特性优越性和背后google强大靠山什么的就不多说了。golang的官方提供了多个平台上的二进制安装包，遗憾的是并非没有发布ARM平台的二进制安装包。ARM平台没办法直接从官网下载二进制安装包来安装，好在golang是支持多平台并且开源的语言，因此可以通过直接在ARM平台上编译源代码来安装。整个过程主要包括编译工具配置、获取golang源代码、设置golang编译环境变量、编译、配置golang运行环境变量等步骤。

注：本文选用树莓派做测试，因为树莓派是基于ARM平台的。

## 1、编译工具配置 ##

据说下个版本的golang编译工具要使用golang自己来写，但目前还是使用C编译工具的。因此，首先要配置好C编译工具：

1.1 在Ubuntu或Debian平台上可以使用`sudo apt-get install gcc libc6-dev`命令安装，树莓派的RaspBian系统是基于Debian修改的，所以可以使用这种方法安装。

1.2 在RedHat或Centos 6平台上可以使用`sudo yum install gcc libc-devel`命令安装。

安装完成后可以输入 `gcc --version`命令验证是否成功安装。


## 2、获取golang源代码 ##

**2.1 直接从官网下载源代码压缩包。**

golang官网提供golang的源代码压缩包，可以直接下载，最新的1.4.1版本源代码链接：`https://storage.googleapis.com/golang/go1.4.1.src.tar.gz`



**2.2 使用git工具获取。**

golang使用git版本管理工具，也可以使用git获取golang源代码。推荐使用这个方法，因为以后可以随时获取最新的golang源代码。

2.2.1 首先确认ARM平台上已经安装了git工具，可以使用git --version命令确认。一般linux平台都安装了git，没有的话可以自行安装，不同平台的安装方法可以参考：`http://git-scm.com/download/linux`

2.2.2 克隆远程golang的git仓库到本地

在终端cd到你想要安装golang的目录，确保该目录下没有名为go的目录。然后以下命令获取代码仓库：

    git clone https://go.googlesource.com/go

大陆地区可能会获取失败，在不翻墙的情况下我试了几次都没成功，原因大家都懂的。好在google已经将golang也托管到github上面，所以也可以通过下面命令获取：

    git clone https://github.com/golang/go.git

视网络情况，下载可能需要不少时间。我2M的带宽花了将近两个小时才下载完，虽然整个项目不过几十兆= =

下载完成后，可以看到目录下多了一个go目录，里面即为golang的源代码，在终端上执行cd go命令进入该目录。

执行下面命令检出go1.4.1版本的源代码，因为现在已经有新的代码提交上去了，最新的代码可能不是最稳定的：

    git checkout go1.4.1

至此，最新1.4.1发行版的源代码获取完毕


## 3、设置golang的编译环境变量 ##

主要有GOROOT、GOOS、GOARCH、GOARM四个环境变量需要设置，先解释四个环境变量的意义。


**3.1 GOROOT**

主要代表golang树结构目录的路径，也就是上面git检出的go目录。一般可以不用设置这个环境变量，因为编译的时候默认会以go目录下src子目录中的all.bash脚本运行时的父目录作为GOROOT的值。为了保险起见，可以直接设置为go目录的路径。


**3.2 GOOS和GOARCH**

分别代表编译的目标系统和平台，可选值如下：

GOOS	GOARCH
darwin	386
darwin	amd64
dragonfly	386
dragonfly	amd64
freebsd	386
freebsd	amd64
freebsd	arm
linux	386
linux	amd64
linux	arm
netbsd	386
netbsd	amd64
netbsd	arm
openbsd	386
openbsd	amd64
plan9	386
plan9	amd64
solaris	amd64
windows	386
windows	amd64

需要注意的是这两个值代表的是目标系统和平台，而不是编译源代码的系统和平台。树莓派的RaspBian是linux系统，所以这些GOOS设置为linux，GOARCH设置为arm。

**3.3 GOARM**

表示使用的浮点运算协处理器版本号，只对arm平台有用，可选值有5，6，7。如果是在目标平台上编译源代码，这个值可以不设置，它会自动判断需要使用哪一个版本。

总结下来，在树莓派上设置golang的编译环境变量，可编辑$HOME/.bashrc文件，在末尾添加下面内容：

    export GOROOT=/usr/local/go
    export GOOS=linux
    export GOARCH=arm

编辑完后保存，执行`source ~/.bashrc`命令让修改生效。


## 4、编译源代码 ##

环境变量配置完成自后就可以开始编译源代码。在go目录下的src子目录中，主要有all.bash和make.bash两个脚本（另外还有两个all.bat和make.bat脚本适用于window平台）。编译实际上就是执行其中一个脚本，两者的区别在于all.bash在编译完成后还会执行一些测试套件。如果希望只编译不测试，可以运行make.bash脚本。使用cd命令进入go下src目录，执行`./all.bash`或者`./make.bash`命令即可开始编译。由于硬件情况不同，编译耗费的时间不同。在我的B型树莓派编译过程花费了将近半个小时，编译完成后执行的测试套件又花费了差不多一个小时，总共花费了一个半小时左右。


## 5、配置golang运行环境变量 ##

编译完成后，go目录下会生成bin目录，里面就是go的运行脚本。为了以后使用方法，可以将这个bin路径添加到PATH环境变量中。同样编辑~/.bashrc文件，因为前面设置过GOROOT环境变量指向go目录了，所以只需要在末尾加上

    export PATH=$PATH:$GOROOT/bin

保存后同样执行`source ~/.bashrc`命令让环境变量生效。


至此，golang源代码编译安装成功。执行`go version`应该就能看到当前golang的版本信息，表示编译安装成功。

另外还有一个比较重要的GOPATH环境变量需要设置，等有时间再讲讲吧。

参考官方文档：`https://golang.org/doc/install/source`