我是个刚学GO的小白，刚一使用就喜欢上了goland的界面和快捷键。

但是我在运行的时候，配置出错了，也就是gopath，而我根据网上的资料也没搞定。

群里有大佬说了解一下gomod，然后我就查了一下度娘

大概意思就是，如果你用了gomod，那么你的项目就不在局限在那个gopath文件夹内了，你可以在任何位置创建项目并且运行。

第一步，保证你的golang的版本是1.11或以上

第二步，在你的goland中，创建一个项目

我创建了一个lalala的项目，然后选择你的项目，点击左下角的Terminal,输入`go mod init lalala`（注意：lalala可以随意命名，但是我习惯和项目名一致）



出现 go: creating new go.mod: module lalala就说明我成功了

可以去翻译一下，大概意思就是一个新文件go.mod创建成功

这个时候，创建一个go文件，名字随便，我这里使用server.go,代码在下面，直接复制进去就好了，注意package要填main

```
package main
 
import (
    "net/http"
    
    "github.com/labstack/echo"
)
 
func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })
    e.Logger.Fatal(e.Start(":1323"))
}
```

然后在左下角的Terminal,输入 go run server.go，不出意外的话，就会像下图这样。

```
$ go run server.go
go: finding github.com/labstack/echo v3.3.10+incompatible
go: downloading github.com/labstack/echo v3.3.10+incompatible
go: extracting github.com/labstack/echo v3.3.10+incompatible
go: finding github.com/labstack/gommon/color latest
go: finding github.com/labstack/gommon/log latest
go: finding github.com/labstack/gommon v0.2.8
# 此处省略很多行
...
 
   ____    __
  / __/___/ /  ___
 / _// __/ _ \/ _ \
/___/\__/_//_/\___/ v3.3.10-dev
High performance, minimalist Go web framework
https://echo.labstack.com
____________________________________O/_______
                                    O\
⇨ http server started on [::]:1323
```

如果报错了，就去File》Settings》GO》Go Modules(vgo)》

    Proxy=https://goproxy.io

在这里我再分享一个阿里云的代理地址：

    https://mirrors.aliyun.com/goproxy/

也是可以填在Proxy里面的，如果https://goproxy.io不能用可以换这个

最后一步， `go get -u` ,来更新所有依赖。

做完之后，这台电脑就有了gomod的环境，可以在任意位置创建项目，可以使用任意IDE，不再关心配置问题。

参考文章：https://segmentfault.com/a/1190000018536993