# 大厂 Go 编程规范（一）：接口使用 #

## 1、如果希望接口方法修改基础数据，则必须使用指针传递 ##

```
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

var f1 F = S1{}
var f2 F = &S2{}

// f1.f() 无法修改底层数据
// f2.f() 可以修改底层数据，给接口变量 f2 赋值时使用的是对象指针
```

只有方法的接收者是一个指针，才能修改底层数据。无论方法的调用者是否是指针，底层数据能否被修改取决于 “方法的接收者” 是否是指针。上面的 S2 方法接收者是指针,所以可以完成数据的修改。

## 2、方法接收者是值，调用者可以是值也可以是指针，但如果接收者是指针，只能指针调用 ##

```
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

//  下面代码无法通过编译。因为 s2Val 是一个值，而 S2 的 f 方法中没有使用值接收器
//   i = s2Val
```

上面的代码，因为 S2 函数的接收者是指针，则只能通过指针调用。这个其实非常容易理解，对于值接收者，需要的是值，如果直接传值肯定没有问题，如果传递的指针，通过指针隐式转化获取对应的值，然后再调用即可。

## 3、接口编译检测 ##

这个一个好习惯，先看下面的bad case。

```
// 如果 Handler 没有实现 http.Handler，会在运行时报错
type Handler struct {
  // ...
}
func (h *Handler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  ...
}
```

如果我们提前判断，就可以在编译期间提前发现问题了。

```
type Handler struct {
  // ...
}
// 用于触发编译期的接口的合理性检查机制
// 如果 Handler 没有实现 http.Handler，会在编译期报错
var _ http.Handler = (*Handler)(nil)

func (h *Handler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

通过接口转化，便可以检查是否实现对应的接口。如果接收者是值，则可以通过“{}”初始化一个对象用于检测。

```
var _ http.Handler = LogHandler{}
func (h LogHandler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```
