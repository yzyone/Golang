
# golang条件编译 #

## 开篇 ##

golang中没有类似C语言中条件编译的写法，比如在C代码中可以使用如下语法做一些条件编译，结合宏定义来使用可以实现诸如按需编译release和debug版本代码的需求

```
#ifndef
#define
...

#end
```

但是golang支持两种条件编译方式

- 编译标签( build tag)
- 文件后缀

## 编译标签( build tag) ##

在源代码里添加标注，通常称之为编译标签( build tag)，编译标签是在尽量靠近源代码文件顶部的地方用注释的方式添加

go build在构建一个包的时候会读取这个包里的每个源文件并且分析编译便签，这些标签决定了这个源文件是否参与本次编译

编译标签添加的规则（附上原文）：

    a build tag is evaluated as the OR of space-separated options
    each option evaluates as the AND of its comma-separated terms
    each term is an alphanumeric word or, preceded by !, its negation

1). 编译标签由空格分隔的编译选项(options)以"或"的逻辑关系组成
2). 每个编译选项由逗号分隔的条件项以逻辑"与"的关系组成
3). 每个条件项的名字用字母+数字表示，在前面加!表示否定的意思

例子（编译标签要放在源文件顶部）

    // +build darwin freebsd netbsd openbsd

这个将会让这个源文件只能在支持kqueue的BSD系统里编译

一个源文件里可以有多个编译标签，多个编译标签之间是逻辑"与"的关系

    // +build linux darwin
    // +build 386

这个将限制此源文件只能在 linux/386或者darwin/386平台下编译.
除了添加系统相关的tag，还可以自由添加自定义tag达到其它目的。
编译方法:
只需要在go build指令后用-tags指定编译条件即可

    go build -tags mytag1 mytag2

注意：刚开始使用编译标签经常会犯下面这个错误

    // +build !linux
    package mypkg // wrong

这个例子里的编译标签和包的声明之间没有用空行隔开，这样编译标签会被当做包声明的注释而不是编译标签从而被忽略掉

下面这个是正确的标签的书写方式，标签的结尾添加一个空行这样标签就不会当做其他声明的注释

    // +build !linux
    
    package mypkg // correct

用go vet命令也可以检测到这个缺少空行的错误，初期可以用这个命令来避免缺少空行的错误

    % go vet mypkg
    mypkg.go:1: +build comment appears too late in file
    exit status 1

作为参考，下面的例子将licence声明,编译标签和包声明放在一起，请大家注意分辨

% head headspin.go

    // Copyright 2013 Way out enterprises. All rights reserved.
    // Use of this source code is governed by a BSD-style
    // license that can be found in the LICENSE file.
    
    // +build someos someotheros thirdos,!amd64
    
    // Package headspin implements calculates numbers so large
    // they will make your head spin.
    package headspin

## 文件后缀 ##

这个方法通过改变文件名的后缀来提供条件编译，这种方案比编译标签要简单，go/build可以在不读取源文件的情况下就可以决定哪些文件不需要参与编译。文件命名约定可以在go/build 包里找到详细的说明，简单来说如果你的源文件包含后缀：**_GOOS.go**，那么这个源文件只会在这个平台下编译，**_GOARCH.go**也是如此。这两个后缀可以结合在一起使用，但是要注意顺序：**_GOOS_GOARCH.go**， 不能反过来用：**_GOARCH_GOOS.go**.
例子如下：

	mypkg_freebsd_arm.go // only builds on freebsd/arm systems
	mypkg_plan9.go       // only builds on plan9

## 编译标签和文件后缀的选择 ##

编译标签和文件后缀的功能上有重叠，例如一个文件名：mypkg_linux.go包含了// +build linux将会出现冗余

通常情况下，如果源文件与平台或者cpu架构完全匹配，那么用文件后缀，例如：

	mypkg_linux.go         // only builds on linux systems
	mypkg_windows_amd64.go // only builds on windows 64bit platforms

相反，如果满足以下任何条件，那么使用编译标签：

- 这个源文件可以在超过一个平台或者超过一个cpu架构下可以使用
- 需要去除指定平台
- 有一些自定义的编译条件