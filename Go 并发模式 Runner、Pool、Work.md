# Go 并发模式 Runner、Pool、Work #

【导读】本文中作者整理了一些 go 语言项目中常用的并发模式。

Go 语言实现的三种常用并发模式；这些模式可以在实际生产应用中合理使用，免去了我们造轮子的过程。

- Runner
- Pool
- Work

下面的代码参考自 《Go 语言实战》

**Runner 定时任务**

Runner 用于展示如何使用通道来监视程序的执行时间，如果程序运行时间太长，也可以用 runner 包来终止程序。在设计上，可以实现以下几点

- 程序可以在分配的时间内完成工作，正常终止；
- 程序没有及时完成工作，“自杀”；
- 接收到操作系统发送的中断事件，程序立刻试图清理状态并停止工作。

这个 Runner 模式还是很有代表性的，他能把（任务队列，超时，系统中断信号）等结合起来形成一项定时任务。任何一个条件满足触发，程序就结束了。

```
import (
 "errors"
 "os"
 "os/signal"
 "time"
)

//Runner 在给定的超时时间内执行一组任务
// 并且在操作系统发送中断信号时结束这些任务
type Runner struct {
 //从操作系统发送信号
 interrupt chan os.Signal
 //报告处理任务已完成
 complete chan error
 //报告处理任务已经超时
 timeout <-chan time.Time
 //持有一组以索引为顺序的依次执行的以 int 类型 id 为参数的函数
 tasks []func(id int)
}

//统一错误处理

// ErrTimeOut 超时错误信息，会在任务执行超时时返回
var ErrTimeOut = errors.New("received timeout")

// ErrInterrupt 中断错误信号，会在接收到操作系统的事件时返回
var ErrInterrupt = errors.New("received interrupt")

//New 函数返回一个新的准备使用的 Runner，d：自定义分配的时间
func New(d time.Duration) *Runner {
 return &Runner{
  interrupt: make(chan os.Signal, 1),
  complete:  make(chan error),
  //会在另一线程经过时间段 d 后向返回值发送当时的时间。
  timeout: time.After(d),
 }
}

//Add 将一个任务加入到 Runner 中
func (r *Runner) Add(tasks ...func(id int)) {
 r.tasks = append(r.tasks, tasks...)
}

//Start 开始执行所有任务，并监控通道事件
func (r *Runner) Start() error {
 //监控所有的中断信号
 signal.Notify(r.interrupt, os.Interrupt)

 //使用不同的 goroutine 执行不同的任务
 go func() {
  r.complete <- r.run()
 }()

 //使用 select 语句来监控 goroutine 的通信
 select {
 //等待任务完成
 case err := <-r.complete:
  return err
 //任务超时
 case <-r.timeout:
  return ErrTimeOut
 }
}

//执行每一个已注册的任务
func (r *Runner) run() error {
 for id, task := range r.tasks {
  //检测操作系统的中断信号
  if r.gotInterrupt() {
   return ErrInterrupt
  }
  //执行已注册的任务
  task(id)
 }
 return nil
}

//检测是否收到了中断信号
func (r *Runner) gotInterrupt() bool {
 select {
 //当中断事件被触发时
 case <-r.interrupt:
  //停止接收后续的任何信号
  signal.Stop(r.interrupt)
  return true
  //继续执行
 default:
  return false
 }
}
```

这个 Runner 类型声明了 3 个通道，用来辅助管理程序的生命周期，以及用来表示顺序执行的不同任务的函数切片

通道被命名为 complete，因为它被执行任务的 goroutine 用来发送任务已经完成的信号。如果执行任务时发生了错误，会通过这个通道发回一个 error 接口类型的值。如果没有发生错误，会通过这个通道发回一个 nil 值作为 error 接口值。

```
const timeout = 3 * time.Second

func main() {
 log.Println("Starting work.")

 r := New(timeout)
 r.Add(createTask(), createTask(), createTask())

 if err := r.Start(); err != nil {
  switch err {
  case ErrTimeOut:
   log.Println("Terminating due to timeout.")
   os.Exit(1)
  case ErrInterrupt:
   log.Println("Terminating due to interrupt.")
  }
 }

 log.Println("Process ended.")
}

func createTask() func(int) {
 return func(id int) {
  log.Printf("Processor -> Task #%d.", id)
  time.Sleep(time.Duration(id) * time.Second)
 }
}
```

打印结果：

```
2021/11/28 21:35:00 Starting work.
2021/11/28 21:35:00 Processor -> Task #0.
2021/11/28 21:35:00 Processor -> Task #1.
2021/11/28 21:35:01 Processor -> Task #2.
2021/11/28 21:35:03 Process ended.
```

**Pool 缓存池**

缓存池的概念随处可见。

在 Go 1.6 及之后的版本中，标准库里自带了资源池的实现

sync.Pool 的使用方式非常简单：

只需要实现 New 函数即可。对象池中没有对象时，将会调用 New 函数创建。

```
var studentPool = sync.Pool{
    New: func() interface{} { 
        return new(Student) // 例如创建 Student 对象
    },
}
```

取得对象和归还对象

```
stu := studentPool.Get().(*Student)
json.Unmarshal(buf, stu) // 使用线程池的对象
studentPool.Put(stu)
```

Get() 用于从对象池中获取对象，因为返回值是 interface{}，因此需要类型转换。Put() 则是在对象使用完毕后，返回对象池。

**work 一个 goroutine 池**

使用无缓冲的通道来创建一个 goroutine 池，这些 goroutine 执行并控制一组工作，让其并发执行。在这种情况下，使用无缓冲的通道要比随意指定一个缓冲区大小的有缓冲的通道好，因为这个情况下既不需要一个工作队列，也不需要一组 goroutine 配合执行。无缓冲的通道保证两个 goroutine 之间的数据交换。

这种使用无缓冲的通道的方法允许使用者知道什么时候 goroutine 池正在执行工作，而且如果池里的所有 goroutine 都忙，无法接受新的工作的时候，也能及时通过通道来通知调用者。使用无缓冲的通道不会有工作在队列里丢失或者卡住，所有工作都会被处理。

转自：

alsritter.icu/posts/7c5d5c47/