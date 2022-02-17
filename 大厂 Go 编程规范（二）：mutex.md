# 大厂 Go 编程规范（二）：mutex #

mutex 是golang 的互斥锁，可以保障在多协程的情况下，数据访问的安全。

## 1、零值有效 ##

我们并不需要mutex指针

```
mu := new(sync.Mutex)
mu.Lock()
```

直接可以使用mutex的零值。

```
var mu sync.Mutex
mu.Lock()
```

## 2、mutex可见性 ##

go的map 非线程安全，所以我们经常会通过mutex 给map 加一个锁，大家先看一下第一种方式：

```
type SMap struct {
  sync.Mutex

  data map[string]string
}
func (m *SMap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

然后我们看一下第二种方式

```
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

感觉差别不大，有啥区别？

从封装的角度来看，第二种方法更加优秀。因为第一种方式，SMap 中的mutex 是大写的，意味着，外部可以直接调用 lock 和unlock 方法，破坏了内部封装原则，所以方法二更好。

## 3、defer更安全 ##

虽然我们可以通过下面的代码，按照需求unlock

```
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount
```

但上面的代码存在两个问题，一是如果分支太多很容易导致unlock ，二是可读性较差，到处是unlock。所以更加推荐下面的写法

```
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count
```

defer的损耗非常少，大家不必纠结。