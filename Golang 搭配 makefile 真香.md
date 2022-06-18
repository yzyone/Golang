# Golang 搭配 makefile 真香 #

【导读】本文介绍了作者在项目中应用 Makefile的实践。

这篇文章打算跟大家聊聊Makefiles，作为一个后端开发者，熟练掌握Makefiles咖啡可以多喝几口。书归正传

golang内置了很多 go commands 可以帮助我们完成go每个阶段的开发工作，但是很多时候我们需要分享我们的代码给其他人，初次看到我们代码工程的人可能并不知道怎么让它跑起来。当然你也可以通过README.md或者其他方式来告知读者。

但对于那些只想快速构建程序的人来说，使用Makefile很好得抽象了技术细节，当我们看到Makefile文件时自然能想到使用make或者make install来构建程序。从此告别记忆长串的命令疯狂敲键盘 偶尔还会敲错的尴尬场景

比如:

```
go build -o hello hello.go
./hello
```

使用Makefile，我们可以很轻松自定义一个target完成这个任务

```
.PHONY: buildandrun
BIN_FILE=hello

buildandrun:
        @go build -o "${BIN_FILE}" hello.go
        ./"${BIN_FILE}"
```

然后我们就可以用如下命令完成工作了

```
make
./"hello"
hello world
```

我们真正上线构建编译时的命令可能是这样的 ：

```
go install -tags="${BUILD_TAGS}" -ldflags "-X version.version=$(VERSION) -X version.date=$(DATE) -X version.commit=$(COMMIT) -X version.branch=$(BRANCH) -w -s" -gcflags=all="-N -l " ./...
```

装配上Makefile，我们仅仅敲4个字符 make即可，我们开发过程中，不同阶段需要干不同的事儿，

- 清理编译中间目标文件
- 跑测试case
- 检查测试覆盖率
- 执行代码检查 等等

Makefile的goal机制对这种情况进行了很好的抽象，以下是我工作当中的Makefile的配置，虽然不是很复杂但真的很有用。

```
.PHONY: all build clean run check cover lint docker help
BIN_FILE=hello
all: check build
build:
    @go build -o "${BIN_FILE}"
clean:
    @go clean
    rm --force "xx.out"
test:
    @go test
check:
    @go fmt ./
    @go vet ./
cover:
    @go test -coverprofile xx.out
    @go tool cover -html=xx.out
run:
    ./"${BIN_FILE}"
lint:
    golangci-lint run --enable-all
docker:
    @docker build -t leo/hello:latest .
help:
    @echo "make 格式化go代码 并编译生成二进制文件"
    @echo "make build 编译go代码生成二进制文件"
    @echo "make clean 清理中间目标文件"
    @echo "make test 执行测试case"
    @echo "make check 格式化go代码"
    @echo "make cover 检查测试覆盖率"
    @echo "make run 直接运行程序"
    @echo "make lint 执行代码检查"
    @echo "make docker 构建docker镜像"
```

**总结**

使用Makefile来管理我们程序的构建，减少了大量输入、拼写错误，简化构建项目的难度。真实线上环境配合CI/CD更佳，如果你还没有尝试使用Makefile，那真的可以试试。



> 转自：
> 
> zhuanlan.zhihu.com/p/345342203