# Golang处理TCP“粘包”问题 #

【导读】TCP 粘包是什么？go 程序需要规避它吗？本文做了详细介绍。

## 1.什么是粘包？ ##

“粘包”这个说法已经被诟病很久了，既然坊间流传这个说法咱们就沿用吧，关于这个问题比较准确的解释可以参考下面几点：

TCP是流传输协议,是一种面向连接的、可靠的、基于字节流的传输层通信协议
TCP没有包的概念，它只负责传输字节序列，UDP是面向数据报的协议，所以不存在拆包粘包问题
应该由应用层来维护消息和消息的边界，即需要一个应用层协议，比如HTTP
所以，本质上这是一个没有正确使用TCP协议的而产生的问题，有网友说了一句非常形象的话：“打开家里的水龙头， 看着自来水往下流， 然后你告诉我， 看， 自来水粘在一起了， 不是有病？”

## 2.如何解决粘包？ ##

通常来说，一般有下面几种方式：

- 消息长度固定，提前确定包长度，读取的时候也安固定长度读取，适合定长消息包。
- 使用特殊的字符或字符串作为消息的边界，例如 HTTP 协议的 headers 以“\r\n”为字段的分隔符
- 自定义协议，将消息分为消息头和消息体，消息头中包含表示消息总长度

## 3.Golang实战 ##

首先，来看一个存在粘包问题的例子：

**一、Server端：**

```
package main

import (
    "log"
    "net"
    "strings"
)

func main() {
    listen, err := net.Listen("tcp", "127.0.0.1:8888")
    if err != nil {
        panic(err)
    }
    defer listen.Close()

    for {
        conn, err := listen.Accept()
        if err != nil {
            panic(err)
        }
        for {
            data := make([]byte, 10)

            _, err := conn.Read(data)

            if err != nil {
                log.Printf("%s\n", err.Error())
                break
            }

            receive := string(data)
            log.Printf("receive msg: %s\n", receive)

            send := []byte(strings.ToUpper(receive))
            _, err = conn.Write(send)
            if err != nil {
                log.Printf("send msg failed, error: %s\n", err.Error())
            }

            log.Printf("send msg: %s\n", receive)
        }
    }
}
```

简单说一下这段代码，有点socket编程的基础的话应该很容易理解，基本上都是Listen -> Accept -> Read这个套路。

有些人一下子就看出来这个服务有点“问题”，它是同步阻塞的，也就意味着这个服务同一时间只能处理一个连接请求，其实解决这个问题也很简单，得益于Go协程的强大，我们只需要开启一个协程单独处理每一个连接就行了。不过这不是今天的主题，有兴趣的童鞋可以自行研究。

**二、Client端：**

这个服务的功能特别简单，客户端输入什么我就返回什么，客户端的话，这里我使用telnet来演示：

```
jwang@jwang:~$ telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
111111
111111
123456
123456
```

当你按回车键的时候telnet会在消息后面自动追加”\r\n“换行符并发送消息！

从代码里面可以看到，在接受消息的时候我们每次读取10个字节的内容输出并返回，如果输入的消息小于等于8（减去换行符）个字符的时候没有问题，但是当我们在telnet里面输入大于10个字符的内容的时候，这些数据的时候会被强行拆开处理。

当然这里有人说了，可不可以一次读多点，然而读多少都会存在这个问题，而且TCP会有缓存区，不一定能够及时把消息发出去，像Nagle优化算法会将多次间隔较小、数据量小的数据，合并成一个大的数据块，然后进行封包，还是会存在问题。

如果我们把这个内容看作是一个业务消息，这个业务消息就被拆分放到下个消息里面处理，必然会产生问题，这就是“粘包”问题的由来。说到底，还是用的人的问题，没有确定好数据边界，如果简单粗暴的读取固定长度的内容，必然会出现问题。

## 4.边界符解决粘包问题 ##

前面说过这个问题，我们可以通过定义一个边界符号解决粘包问题，比如说在上面的例子里面telnet会自动在每一条消息后面追加“\r\n”符号，我们恰好可以利用这点来区分消息。

- 定义一个buffer来临时存放消息
- 从conn里面读取固定字节大小内容，判断当前内容里面有没有分隔符
- 如果没有找到分隔符，把当前内容追加到buffer里面，然后重复第2步
- 如果找到分隔符，把当前内容里面分隔符之前的内容追加到buffer后输出
- 然后重置buffer，把分隔符之后的内容追加到buff，重复第2步
- 不过Go里面提供了一个非常好用的buffer库，为我们节省了很多操作

我们可以使用bufio库里面的NewReader把conn包装一下，然后使用ReadSlice方法读取内容，该方法会一直读直到遇到分隔符，非常简单实用。

**一、Server端：**

```
package main

import (
    "bufio"
    "fmt"
    "net"
)

func main() {
    listen, err := net.Listen("tcp", "127.0.0.1:8888")
    if err != nil {
        panic(err)
    }
    defer listen.Close()

    for {
        conn, err := listen.Accept()
        if err != nil {
            panic(err)
        }
        reader := bufio.NewReader(conn)
        for {
            slice, err := reader.ReadSlice('\n')
            if err != nil {
                continue
            }
            fmt.Printf("%s", slice)
        }
    }
}
```

**二、Client端：**

Client这里可以直接使用telnet，也可以自己写一个，代码如下：

```
package main

import (
    "log"
    "net"
    "strconv"
    "testing"
    "time"
)

func Test(t *testing.T) {
    conn, err := net.Dial("tcp", "127.0.0.1:8888")
    if err != nil {
        log.Println("dial error:", err)
        return
    }
    defer conn.Close()
    i := 0
    for {
        var err error
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 77777\n"))
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 88888\n"))
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 555555555555555555555555555555555555555555\n"))
        if err != nil {
            panic(err)
        }
        time.Sleep(time.Second * 1)
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 123456\n"))
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 123456\n"))
        if err != nil {
            panic(err)
        }
        time.Sleep(time.Second * 1)
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 9999999\n"))
        _, err = conn.Write([]byte(strconv.Itoa(i) + " => 0000000000000000000000000000000000000000000\n"))
        if err != nil {
            panic(err)
        }
        i++
    }
}
```

如果要说缺点，这种方式主要存在2点，第一点是分隔符的选择问题，如果需要传输的消息包含分隔符，那就需要提前做转义处理。第二点就是性能问题，如果消息体特别大，每次查找分隔符的位置的话肯定会有一点消耗。

## 5.在头部放入信息长度 ##

目前应用最广泛的是在消息的头部添加数据包长度，接收方根据消息长度进行接收；在一条TCP连接上，数据的流式传输在接收缓冲区里是有序的，其主要的问题就是第一个包的包尾与第二个包的包头共存接收缓冲区，所以根据长度读取是十分合适的。

**一、Server端：**

```
package main

import (
    "bufio"
    "bytes"
    "encoding/binary"
    "fmt"
    "net"
)

func main() {
    listen, err := net.Listen("tcp", "127.0.0.1:8888")
    if err != nil {
        panic(err)
    }
    defer listen.Close()

    for {
        conn, err := listen.Accept()
        if err != nil {
            panic(err)
        }
        reader := bufio.NewReader(conn)
        for {
            //前4个字节表示数据长度
            peek, err := reader.Peek(4)
            if err != nil {
                continue
            }
            buffer := bytes.NewBuffer(peek)
            //读取数据长度
            var length int32
            err = binary.Read(buffer, binary.BigEndian, &length)
            if err != nil {
                continue
            }
            //Buffered 返回缓存中未读取的数据的长度,如果缓存区的数据小于总长度，则意味着数据不完整
            if int32(reader.Buffered()) < length+4 {
                continue
            }
            //从缓存区读取大小为数据长度的数据
            data := make([]byte, length+4)
            _, err = reader.Read(data)
            if err != nil {
                continue
            }
            fmt.Printf("receive data: %s\n", data[4:])
        }
    }
}
```

**二、Client端：**

需要注意的是发送数据的编码，这里使用了Go的binary库，先写入4个字节的头，再写入消息主体，最后一起发送过去。

```
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
    "log"
    "net"
    "testing"
    "time"
)

func Test(t *testing.T) {
    conn, err := net.Dial("tcp", "127.0.0.1:8888")
    if err != nil {
        log.Println("dial error:", err)
        return
    }
    defer conn.Close()
    for {
        data, _ := Encode("123456789")
        _, err := conn.Write(data)
        data, _ = Encode("888888888")
        _, err = conn.Write(data)
        time.Sleep(time.Second * 1)
        data, _ = Encode("777777777")
        _, err = conn.Write(data)
        data, _ = Encode("123456789")
        _, err = conn.Write(data)
        time.Sleep(time.Second * 1)
        fmt.Println(err)
    }
}
func Encode(message string) ([]byte, error) {
    // 读取消息的长度
    var length = int32(len(message))
    var pkg = new(bytes.Buffer)
    // 写入消息头
    err := binary.Write(pkg, binary.BigEndian, length)
    if err != nil {
        return nil, err
    }
    // 写入消息实体
    err = binary.Write(pkg, binary.BigEndian, []byte(message))
    if err != nil {
        return nil, err
    }
    return pkg.Bytes(), nil
}
```

## 6.总结 ##

世界上本没有“粘包”，只不过是少数人没有正确处理TCP数据边界问题，成熟的应用层协议（http、ssh）都不会存在这个问题。但是如果你使用纯TCP自定义协议，那就需要自己处理好了。



转自：

wangbjun.site/2019/coding/golang/golang-tcp-package.html