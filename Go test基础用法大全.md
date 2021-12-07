# Go test基础用法大全 #

【导读】go 语言的 test 命令有很多参数，怎么利用 test 命令和它提供的参数，又能做到什么？本文做了详细解读。

当直接使用IDE进行单元测试时，有没有好奇它时如何实现的？比如GoLand写的测试用例。

所有的代码都需要写测试用例。这不仅仅是对自己的代码负责，也是对别人的负责。

最近工作中使用glog这个库，因为它对外提供的方法都很简单，想封装处理一下。但却遇到了点麻烦：这个包需要在命令行传递log_dir参数，来指定日志文件的路径。

所以，正常运行的话，首先需要编译可执行文件，然后命令行指定参数执行。如下示例：

    go build main.go
    ./main -log_dir="/data"//当前目录作为日志输出目录

但在go test的时候，如何指定这个参数了？

**Test**

调查发现，发现go test也可以生成可执行文件。需要使用-c来指定。示例如下：

	go test -c param_test_dir   //最后一个参数是待测试的目录

执行后就会发现：这样的做法，会运行所有的Test用例。如何仅仅执行某一个测试用例了（编译器到底是如何做到的？）。

这里有另一个属性-run，用来指定执行的测试用例的匹配模式。举个例子：

```
func TestGetRootLogger(t *testing.T) {
 writeLog("测试")
}

func TestGetRootLogger2(t *testing.T) {
 writeLog("测试 2")
}
```

当我在命令行明确匹配执行Logger2，运行的时候确实仅仅执行该测试用例

	go test -v -run Logger2 ./util/     //-v 表示 verbose，输出相信信息

但是，我发现，在指定了c参数之后，run参数无法生效！这样的话，还真是没有好的办法来处理这种情况。

**option**

	-timeout

默认go test运行超过10m会发生panic。如果需要运行更长的时间，需要明确指定。将timeout指定为 0，用于忽略时间限制。

	nohup go test -v -timeout 0 -run TestGetRange . > log.txt

**使用map的测试**

可以结合使用闭包，设置期望值，来写测试用例。Run函数内部是阻塞的，所以TestSum方法依次执行测试。

同时testSumFunc返回了test方法使用了闭包的特性，对返回函数内部的值是无法确定的。

```
func TestSum(t *testing.T) {
 t.Run("A", testSumFunc([]int{1, 2, 3}, 7))
 t.Run("B", testSumFunc([]int{2, 3, 4}, 8))
}

func Sum(numbers []int) int {
 total := 0
 for _, v := range numbers {
  total += v
 }

 return total
}

func testSumFunc(numbers []int, expected int) func(t *testing.T) {
 return func(t *testing.T) {
  actual := Sum(numbers)
  if actual != expected {
   t.Error(fmt.Sprintf("Expected the sum of %v to be %d but instead got %d!", numbers, expected, actual))
  }
 }
}
```

**Main**

非常简单，看如下示例。这样在执行任何test case时都首先执行准备，在测试用例执行完毕后，会运行清理工作。需要特别说明的是：flag.Parse()以及os.Exit(m.Run())是不可缺少的两步。

```
func TestMain(m *testing.M) {
    //准备工作
 fmt.Println("start prepare")

 flag.Parse()
 exitCode := m.Run()
    
    //清理工作
 fmt.Println("prepare to clean")
 
 os.Exit(exitCode)
}
```

**性能测试pprof**

定位服务是否存在资源泄漏或者滥用API的行为，光靠review代码是不行的，最好能借助工具。

**Profile**

引用 godoc for pprof 描述：

```
A Profile is a collection of stack traces showing the call sequences that led to instances of a particular event, such as allocation. Packages can create and maintain their own profiles; the most common use is for tracking resources that must be explicitly closed, such as files or network connections.
```

性能测试涉及如下方面：

- CPU Profiling：CPU分析，按照一定的频率采集所监听的应用程序CPU（含寄存器）的使用情况，可确定应用程序在主动消耗CPU 周期时花费时间的位置
- Memory Profiling：内存分析，在应用程序进行堆分配时记录堆栈跟踪，用于监视当前和历史内存使用情况，以及检查内存泄漏
- Block Profiling：阻塞分析，记录 goroutine 阻塞等待同步（包括定时器通道）的位置
- Mutex Profiling：互斥锁分析，报告互斥锁的竞争情况
- 在程序中引入如下包，便可以通过 web 方式查看性能情况，访问的路径为：/debug/pprof/，该路径下会显示多个查看项。该路径下还有其他子页面。


_ "net/http/pprof"

关于/debug/pprof/下的子页面：

- $HOST/debug/pprof/profile
- $HOST/debug/pprof/block
- $HOST/debug/pprof/goroutine
- $HOST/debug/pprof/heap
- $HOST/debug/pprof/mutex
- $HOST/debug/pprof/threadcreate

**在终端查看性能**

只要服务器在启动时，引入pprof包，便可在终端获取profile文件。如下所示：

	pprof -seconds=10 http://192.168.77.77:3900/debug/pprof/profile

如果获取到cpu.prof文件，可以通过如下命令可视化查看运行结果，这是另一种格式的火焰图，也是挺帅的：

```
## 通过在浏览器中 localhost:1313 可以在 web 端查看
## 
pprof -http=:1313 cpu.prof

## 或直接在终端查看
go tool pprof cpu.prof
$ web | top
```

**Benchmark测试**

基本用法：

```
func BenchmarkBadgeRelationMapper_GetCountByUid(b *testing.B) {

 count := 0
 for i := 0; i < b.N; i++ {
  count++
 }
 fmt.Println("total:", count)
}
```

bench测试输出结果，函数体被重复执行了 6 次，并对b.N的值做了调整：

```
total: 1
total: 100
total: 10000
total: 1000000
total: 100000000
total: 1000000000
1000000000          0.598 ns/op
```

并发用法：

```
func BenchmarkBadgeRelationMapper_GetCountByUid(b *testing.B) {

 count := 0
 b.RunParallel(func(pb *testing.PB) {
  for pb.Next() {
   count++
  }
 })

 fmt.Println("total:", count)
}
```

输出的结果：

```
total: 1
total: 100
total: 6336
total: 306207
total: 34221963
total: 129821900
378966799          2.94 ns/op
```

在并行用法中，b.N被RunParallel接管。

简单分析一下源码

核心思路在于Next方法，通过atomic.AddUint64并发安全的操作pb.globalN，pb.cache用来存储该goroutine执行的次数。当某个goroutine计算到pb.bN<=n<=pb.bN+pb.grain时，虽然程序迭代次数已经完全超过b.N，但还是会让它继续执行。

```
// Next reports whether there are more iterations to execute.
func (pb *PB) Next() bool {
 if pb.cache == 0 {
  n := atomic.AddUint64(pb.globalN, pb.grain)
  if n <= pb.bN {
   pb.cache = pb.grain
  } else if n < pb.bN+pb.grain {
   pb.cache = pb.bN + pb.grain - n
  } else {
   return false
  }
 }
 pb.cache--
 return true
}
```

**regular expression**

先列举参考的example。在我们要运行特定case时，可以通过指定正则表达式来实现：

```
// -bench takes a regular expression that matches the names of the benchmarks you want to run
go test -bench=. ./examples/fib/

// -run flag with a regex that matches nothing
go test -run=^$
```

关于如何运行Benchmark测试，默认执行go test并不会执行Benchmark，需要在命令行明确加上-bench=标记，它接受一个表达式作为参数，匹配基准测试的函数，. 表示运行所有基准测试。

```
go test -bench=.

// 明确指定要运行哪个测试，传递一个正则表达式给 run 属性，XXX=BenchmarkReceiveGift_GetGiftReceiveList
go test -run=XXX -bench=.
```

默认情况下，benchmark最小运行时长为1s。如果benchmark函数执行返回，但1s的时间还没有结束，b.N会根据某种机制依次递增。可以通过参数-benchtime=20s来改变这种行为。

还有一个参数：benchmem。可以提供每次操作分配内存的次数，以及每次操作分配的字节数。

	go test -bench=Fib40 -benchtime=20s

**Run Example**

获取线上的pprof数据到本地，这里是另一个工具：

	go-torch -u http://192.168.77.77:3900/debug/pprof/profile -t 10

在Go 代码调优利器-火焰图这篇文章中，对例子介绍的挺精彩的。

```
## 对函数 GetGiftReceiveList 进行 Benchmark 测试 因为只想压测 GetGiftReceiveList 这个函数
## 所以指定了 run 参数
go test -bench . -run=GetGiftReceiveList -benchmem -cpuprofile prof.cpu

## 其中 present.test 是压测的二进制文件，prof.cpu 也是生产的文件
## (pprof) top10
## (pprof) list Marshal 单独查看这个函数的耗时，这里应该是正则匹配的
go tool pprof present.test prof.cpu

## 查看内存使用情况
go test -bench . -benchmem -memprofile prof.mem
go tool pprof --alloc_objects  present.test prof.mem
```

覆盖率

跟执行go test不同的是，需要多加一个参数-coverprofile, 所以完整的命令：

	go test -v -coverprofile=c.out

生成报告有 go 为我们提供的工具，使用

	go tool cover -html=c.out -o=tag.html

即可生成一个名字为 tag.html 的 HTML 格式的测试覆盖率报告，这里有详细的信息告诉我们哪一行代码测试到了，哪一行代码没有测试到。

**火焰图**

学习了解火焰图，分析函数调用栈的信息。主要是相关的工具：

```
## tool1
git clone https://github.com/brendangregg/FlameGraph.git
cp flamegraph.pl /usr/local/bin
flamegraph.pl -h

go get -v github.com/uber/go-torch
go-torch -h
```

转自：

neojos.com/blog/2018/05-02-go-test%E5%9F%BA%E7%A1%80%E7%94%A8%E6%B3%95/

参考链接：https://mp.weixin.qq.com/s/-aFPd3EoVVF6-MWZOfKlXA