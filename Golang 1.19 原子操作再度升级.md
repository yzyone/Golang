# Golang 1.19 原子操作再度升级

8月2号，Go 1.19 终于发布，这次的更新包含了对于泛型带来的诸多问题修复，提升了泛型性能（据官方描述性能优化了 20%），以及内存模型与C, C++, Java, Rust 的对齐，还有我们今天的主角：sync/atomic 的新类型。

感兴趣的同学可以看下官方的 release blog[1]，release note[2] 以及 下载安装[3]，体验一下新版本带来的特性。

今天我们来了解一下加入 sync/atomic 大家庭的几个新类型：Bool, Int32, Int64, Uint32, Uint64, Uintptr, Pointer.

## atomic.Value 回顾

如果我们做时光机回到 Go 1.18[4] 甚至一路追溯到 Go 1.4 版本，你会发现 atomic 包虽然提供了很多函数，但只有一个 Type，那就是 atomic.Value。这一节我们回顾一下 atomic.Value 提供的能力。

> A Value provides an atomic load and store of a consistently typed value. The zero value for a Value returns nil from Load. Once Store has been called, a Value must not be copied.
> 
> A Value must not be copied after first use.

回顾一下我们的此前的原子操作解读[5]，所谓 atomic.Value 就是一个容器，可以被用来“原子地”存储和加载任意的值，并且是开箱即用的。

### 使用场景

atomic.Value 的两个经典使用场景：

1. 周期性更新配置，并提供给多个协程

```
package main  

import (  
 "sync/atomic"  
 "time"  
)  

func loadConfig() map[string]string {  
 return make(map[string]string)  
}  

func requests() chan int {  
 return make(chan int)  
}  

func main() {  
 var config atomic.Value // holds current server configuration  
 // Create initial config value and store into config.  
 config.Store(loadConfig())  
 go func() {  
  // Reload config every 10 seconds  
  // and update config value with the new version.  
  for {  
   time.Sleep(10 * time.Second)  
   config.Store(loadConfig())  
  }  
 }()  
 // Create worker goroutines that handle incoming requests  
 // using the latest config value.  
 for i := 0; i < 10; i++ {  
  go func() {  
   for r := range requests() {  
    c := config.Load()  
    // Handle request r using config c.  
    _, _ = r, c  
   }  
  }()  
 }  
}  
```

关注 main 函数，我们只需要用 `var config atomic.Value` 声明一个 atomic.Value 出来，开箱即用。然后用 `Store` 存入【值】，随后开启一个 goroutine 异步来定时更新 config 中存储的值即可。

这里其实真正用到的就三点：

- 声明 atomic.Value；

- 调用 Store 来更新容器中存储的值；

- 调用 Load 来获取到存储的值。
2. 针对读多写少场景的 Copy-On-Write

```
package main  

import (  
 "sync"  
 "sync/atomic"  
)  

func main() {  
 type Map map[string]string  
 var m atomic.Value  
 m.Store(make(Map))  
 var mu sync.Mutex // used only by writers  
 // read function can be used to read the data without further synchronization  
 read := func(key string) (val string) {  
  m1 := m.Load().(Map)  
  return m1[key]  
 }  
 // insert function can be used to update the data without further synchronization  
 insert := func(key, val string) {  
  mu.Lock() // synchronize with other potential writers  
  defer mu.Unlock()  
  m1 := m.Load().(Map) // load current value of the data structure  
  m2 := make(Map)      // create a new value  
  for k, v := range m1 {  
   m2[k] = v // copy all data from the current object to the new one  
  }  
  m2[key] = val // do the update that we need  
  m.Store(m2)   // atomically replace the current object with the new one  
  // At this point all new readers start working with the new version.  
  // The old version will be garbage collected once the existing readers  
  // (if any) are done with it.  
 }  
 _, _ = read, insert  
}  
```

这里维护了一个 Map，每次插入 key 时直接创建一个新的 Map，通过 atomic.Value.Store 赋值回来。读的时候用 atomic.Value.Load 转换成 Map 即可。

两个例子都只用到了 Load 和 Store，但其实在 Go 1.17 版本就已经新增了两个方法：

- `CompareAndSwap`：经典 CAS，传入的 old 和 new 两个 interface{} 必须要是同一个类型（nil也不可以），否则会 panic

- `Swap`：将一个新的 interface{} 存入 Value，并返回此前存储的值，若Value为空，则返回 nil。和 CAS 同样的也必须是类型一致。

下面是目前 atomic.Value 支持的四种方法，应对绝大部分业务场景是绰绰有余了。

![图片](https://mmbiz.qpic.cn/mmbiz/IgylNib7ZE2LdJ1tpt4BFnf1nQwFicaZyUpbaPAaE3g13O9oYWYxUeHHo7zLVKbcnf5CfdBlMZge6E6RVWhiaA9rA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

## 新增基础类型

好，进入今天的正题，Go 1.19 当中引入的几个新类型，从官方的解释来看有三个好处：

> - These types hide the underlying values so that all accesses are forced to use the atomic APIs.
> 
> - Pointer also avoids the need to convert to unsafe.Pointer at call sites.
> 
> - Int64 and Uint64 are automatically aligned to 64-bit boundaries in structs and allocated data, even on 32-bit systems.

整体定位上还是希望引入了新类型，来为开发者提供更多便利，理论上讲不依赖这些类型，其实我们用 atomic 下的函数也能搞定，但会比较麻烦罢了。

这几个新类型中， Bool, Int32, Int64, Uint32, Uint64 都是针对基础数据类型进行的封装。这一节我们先来看看基础类型提供了什么能力，后面我们再来看重头戏：Uintptr 和 Pointer。

1. Bool
- func (x *Bool) CompareAndSwap(old, new bool) (swapped bool)

- func (x *Bool) Load() bool

- func (x *Bool) Store(val bool)

- func (x *Bool) Swap(new bool) (old bool)
2. Int32
- func (x *Int32) Add(delta int32) (new int32)

- func (x *Int32) CompareAndSwap(old, new int32) (swapped bool)

- func (x *Int32) Load() int32

- func (x *Int32) Store(val int32)

- func (x *Int32) Swap(new int32) (old int32)
3. Int64
- func (x *Int64) Add(delta int64) (new int64)

- func (x *Int64) CompareAndSwap(old, new int64) (swapped bool)

- func (x *Int64) Load() int64

- func (x *Int64) Store(val int64)

- func (x *Int64) Swap(new int64) (old int64)
4. Uint32
- func (x *Uint32) Add(delta uint32) (new uint32)

- func (x *Uint32) CompareAndSwap(old, new uint32) (swapped bool)

- func (x *Uint32) Load() uint32

- func (x *Uint32) Store(val uint32)

- func (x *Uint32) Swap(new uint32) (old uint32)
5. Uint64
- func (x *Uint64) Add(delta uint64) (new uint64)

- func (x *Uint64) CompareAndSwap(old, new uint64) (swapped bool)

- func (x *Uint64) Load() uint64

- func (x *Uint64) Store(val uint64)

- func (x *Uint64) Swap(new uint64) (old uint64)

看一下API 我们就能发现，其实和 atomic.Value 非常类似，核心能力都在于这四个接口

- Load

- Store

- Swap

- CompareAndSwap

作用和 atomic.Value 是完全一样的，只是原来我们还需要用 interface{} 来作为媒介进行转化，现在我们可以直接用对应类型了，便捷了很多。

对这些基础类型新增封装，隐含的好处在于：强制使用接口能力来操作。

这一点可能不太好理解，试想一下，如果此刻没有这些封装，我们依然想对于一个 int32 来做上面的【原子四操作】，会怎么处理？

很简单，我们会用 atomic 提供的函数：

- `func LoadInt32(addr *int32) (val int32)`

- `func StoreInt32(addr *int32, val int32)`

- `func SwapInt32(addr *int32, new int32) (old int32)`

- `func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)`

功能上，逻辑上是完全一致的，我直接用就 ok 了，何必还要用 atomic.Int32 提供的方法呢？

原因在于，如果你直接用 int32 来操作，你可以针对这个 int32 变量做任何事。如果你经验丰富，意识准确，用法完全正确，没问题，不用切过来。

但是，但是，一旦某天你状态不好，或组里其他人改你的代码，在某个不经意的地方拿这个 int32 变量的地址去干了其他看似人畜无害的事（这很难完全避免），程序可能因此出现意想不到的反应，因为表面上用了【原子操作】，但会存在其他用法的干扰，这就很讲究开发者的意识和素养了。

所以，Golang 官方提出 atomic 的几个基础类型，就是为了约束，既然是个 atomic.Int32，那对不起，你必须用我提供的 API 对其进行操作，保证全是原子的，这样能避免很多无意识带来的 bug。

当然，除了经典4方法，结合具体的原子类型，Golang 还添加了一些补充方法，Int32, Uint32, Int64, Uint64 都扩充了 `Add(delta) new` 的原子操作用来实现加法。这也是另一个层面的【约束】，接口来提供【加】的能力，不要自己用原始的 int32 类型来加，保证所有操作都是原子API 提供的能力。

## 新增的 Uintptr 类型

还记得 uintpr 么？我们在此前 Golang 中的 unsafe.Pointer 和 uintptr[6] 里提到过，不熟悉的同学建议温习一下。

简单说，uintptr 是一个可以存储任何指针地址的【整型】，注意 uintptr 指的是具体的内存地址，不是个指针，没有指针的语义。

我们可以将 uintptr 转换成 unsafe.Pointer (一个可以指向任何一种类型的【指针】)

Golang 1.19 新增的 atomic.Uintptr 类型支持的 API 是这几种：

- func (x *Uintptr) Load() uintptr

- func (x *Uintptr) Store(val uintptr)

- func (x *Uintptr) Swap(new uintptr) (old uintptr)

- func (x *Uintptr) CompareAndSwap(old, new uintptr) (swapped bool)

- func (x *Uintptr) Add(delta uintptr) (new uintptr)

跟上面提到的 atomic.Int32, atomic.Uint32 以及 atomic.Int64, atomic.Uint64 是一样的，在经典四能力之外新增了 Add 的能力。

这也是进一步提醒大家，uintptr 就是个整数，只是用来存内存地址，但毕竟还是个整数，所以能支持【加法】，而且很多时候我们会依赖这个加法，计算出另一个关联实体的内存地址（比如 slice 中的某个元素的地址）

# 新增的 Pointer 类型

好饭留到最后，其实这次新增的类型中，个人认为最有用的还是 atomic.Pointer。

基础类型的封装的确能避坑，但只是小优化。Pointer 类型的引进，是在泛型能力之上，对 atomic.Value 的很好的补充。

先看定义：

> A Pointer is an atomic pointer of type *T. The zero value is a nil *T.

```
type Pointer[T any] struct {  
 // contains filtered or unexported fields  
}  
```

注意，泛型来了，要消除 atomic.Value 每次从 interface{} 来断言的复杂度，每一个 atomic.Pointer 都需要指明类型。我们比较一下 atomic.Value 和 atomic.Pointer 初始化的区别：

```
type Map map[string]string  

var m atomic.Value  
m.Store(make(Map))  
m1 := m.Load().(Map)  

m := atomic.Pointer[Map]{}  
newMap := make(Map)  
m.Store(&newMap)  
m1 := m.Load()  
```

在 atomic.Value 的用法下，声明之后开箱即用，但是没有类型。而在 atomic.Pointer 声明时你就需要指明【类型】，带了类型之后，我们就可以直接 Store 和 Load 指定类型的变量指针了，而不用每次转 interface{}。

Pointer 支持的 API 和其他类型一样，还是经典的【原子4能力】，只不过这里操作的对象都是【指针】：

- func (x *Pointer[T]) Load() *T

- func (x *Pointer[T]) Store(val *T)

- func (x *Pointer[T]) Swap(new *T) (old *T)

- func (x *Pointer[T]) CompareAndSwap(old, new *T) (swapped bool)

Swap 和 Store 其实基本功能是一样的，只是 Swap 会返回此前存储在 Pointer 中的指针，而 Store 没有返回值罢了。

### 实战用法

我们来看一个完整的 demo

```
package main  

import (  
 "fmt"  
 "net"  
 "sync/atomic"  
)  

type ServerConn struct {  
 Connection net.Conn  
 ID string  
 Open bool  
}  

func main() {  
 p := atomic.Pointer[ServerConn]{}  
 s := ServerConn{ ID : "first_conn"}  
 p.Store( &s )  
 fmt.Println(p.Load()) // Will display value stored.  
}  
```

这里我们基于 ServerConn 类型来创建一个 atomic.Pointer，随后就可以用来 Store 指针，并通过 Load 来获取一个 *ServerConn。

那么，假设我们要周期性地更新这个 ServerConn，可以这样操作：

```
...  
func ShowConnection(p * atomic.Pointer[ServerConn]){  
for {  
  time.Sleep(10 * time.Second)  
  fmt.Println(p, p.Load())  
 }  
}  

func main() {  
 c := make(chan bool)  
 p := atomic.Pointer[ServerConn]{}  
 s := ServerConn{ ID : "first_conn"}  
 p.Store( &s )  

 go ShowConnection(&p)  

 go func(){  
   for {  
    time.Sleep(15 * time.Second)  
    newConn := ServerConn{ ID : "new_conn"}  
    p.Swap(&newConn)  
   }  
  }()  
  <- c  
}  
```

我们在一个 goroutine 里执行 ShowConnection，打印连接信息。在另一个 goroutine 里每过 15 秒就 swap 一个新的连接进去。用一个 bool channel 来控制主 goroutine 的退出。

## 争议：指针赋值的原子性

我们之前[7]也提到过，Golang 目前的指针赋值可以认为是原子的，那是不是不用 atomic.Pointer 也行？

并不是！由于CPU 三级缓存，指令重排，可见性等问题，从业务视角看，我们要求的【原子性】不是简单的【写入原子】即可。而是不仅仅要写，还要让我们能看到，能读到。atomic 包提供的方法会提供内存屏障的功能，所以，atomic 不仅仅可以保证赋值的数据完整性，还能保证数据的可见性，一旦一个核更新了该地址的值，其它处理器总是能读取到它的最新值。

还是那句话，遵循官方规范即可，有一些很 trick 的方式有可能拿到更好的性能，但这些 trick 依赖的上下文是可能会变的，Golang 保证 1.x 版本的兼容性，但不代表一切都不会变。每当想搞花活的时候，想想 Golang Memory Model 里面那句经典的 advice：

> Programs that modify data being simultaneously accessed by multiple goroutines must serialize such access.
> 
> To serialize access, protect the data with channel operations or other synchronization primitives such as those in the sync and sync/atomic packages.
> 
> If you must read the rest of this document to understand the behavior of your program, you are being too clever.
> 
> Don't be clever.

## 结语

今天我们借着 Go 1.19 对 atomic 包的扩充来回顾了原子操作，并了解了 atomic.Pointer 的使用。原子操作相对于锁和 channel 都是非常轻量级的解决方案，建议大家用好 atomic 包目前提供的能力，对提高程序性能和并发安全性都有帮助。感谢阅读，欢迎在评论区交流！

## 参考资料

- Atomic Pointers in Go 1.19[8]

- atomic 官方文档[9]

### 参考资料

[1]release blog: https://go.dev/blog/go1.19

[2]release note: https://tip.golang.org/doc/go1.19

[3]下载安装: https://go.dev/dl/

[4]Go 1.18: https://pkg.go.dev/sync/atomic@go1.18#pkg-types

[5]原子操作解读: *https://juejin.cn/post/7119437493547565063#heading-4*

[6]Golang 中的 unsafe.Pointer 和 uintptr: *https://juejin.cn/post/7127600972573966373*

[7]之前: *https://juejin.cn/post/7119437493547565063#heading-5*

[8]Atomic Pointers in Go 1.19: https://betterprogramming.pub/atomic-pointers-in-go-1-19-cad312f82d5b

[9]atomic 官方文档: https://pkg.go.dev/sync/atomic#Pointer

> 转自：
> 
> https://juejin.cn/post/7132322169007702047
