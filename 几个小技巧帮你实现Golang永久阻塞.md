# 几个小技巧帮你实现Golang永久阻塞 #

Go 的运行时的当前设计，假定程序员自己负责检测何时终止一个 goroutine 以及何时终止该程序。可以通过调用 os.Exit 或从 main() 函数的返回来以正常方式终止程序。而有时候我们需要的是使程序阻塞在这一行。

几个小技巧帮你实现Golang永久阻塞

转自：juejin.cn/post/6844903598216871943

整理：地鼠文档：www.topgoer.cn

**使用 sync.WaitGroup**

一直等待直到 WaitGroup 等于 0

```
package main

import "sync"

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    wg.Wait()
}
```

**空 select**


select{}是一个没有任何 case 的 select，它会一直阻塞

```
package main

func main() {
    select{}
}
```

**死循环**

虽然能阻塞，但会 100%占用一个 cpu。不建议使用

```
package main

func main() {
    for {}
}
```

**用 sync.Mutex**

一个已经锁了的锁，再锁一次会一直阻塞，这个不建议使用

```
package main

import "sync"

func main() {
    var m sync.Mutex
    m.Lock()
}
```

**os.Signal**

系统信号量，在 go 里面也是个 channel，在收到特定的消息之前一直阻塞

```
package main

import (
    "os"
    "syscall"
    "os/signal"
)

func main() {
    sig := make(chan os.Signal, 2)
    signal.Notify(sig, syscall.SIGTERM, syscall.SIGINT)
    <-sig
}
```

**空 channel 或者 nil channel**

channel 会一直阻塞直到收到消息，nil channel 永远阻塞。

```
package main

func main() {
    c := make(chan struct{})
    <-c
}
```

**总结**

注意上面写的的代码大部分不能直接运行，都会 panic，提示“all goroutines are asleep - deadlock!”，因为 go 的 runtime 会检查你所有的 goroutine 都卡住了， 没有一个要执行。你可以在阻塞代码前面加上一个或多个你自己业务逻辑的 goroutine，这样就不会 deadlock 了。