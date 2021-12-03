
# Go 语言使用 cgo 时的内存管理 #

【导读】本文中作者介绍了使用 cgo 时的关注点和使用 pprof 对 cgo 程序进行排查时的注意点。

**先放结论**

使用 cgo 时：

- 和日常 Go 对象被 gc 管理释放的表现略有不同的是，Go 和 c 代码的类型相互转化传递时，有时需要在调用结束后手动释放内存。
- 有时类型转换伴随着内存拷贝的开销。
- 如果想消除拷贝开销，可以通过unsafe.Pointer获取原始指针进行传递。
- c 代码中的内存泄漏，依然可以使用 valgrind 检查。但是需要注意，像C.CString这种泄漏，valgrind 无法给出泄漏的准确位置。
- go pprof无法检查 c 代码中的内存泄漏。

**引子**

Go 的 cgo 介绍页面 或 源码中的注释文档 有如下例子：

```
package main

// #include <stdio.h>
// #include <stdlib.h>
//
// static void myprint(char* s) {
//   printf("%s\n", s);
// }
import "C"
import "unsafe"

func main() {
  cs := C.CString("Hello from stdio")
  C.myprint(cs)
  C.free(unsafe.Pointer(cs)) // yoko注，去除这行将发生内存泄漏
}
```

从上面例子可以看到，Go 代码中的cs变量在传递给 c 代码使用完成之后，需要调用C.free进行释放。

文档中也对C.CString的释放做了如下强调说明：

```
// Go string to C string
// The C string is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h
// if C.free is needed).
func C.CString(string) *C.char {}

// 翻译成中文：
// C string在C的堆上使用malloc申请。
// 调用者有责任在合适的时候对该字符串进行释放，释放方式可以是调用C.free（调用C.free需包含stdlib.h）
```

另外值得说明的是，以下几种类型转换，都会发生内存拷贝

```
// Go string to C string
func C.CString(string) *C.char {}

// Go []byte slice to C array
// 这个和C.CString一样，也需要手动释放申请的内存
func C.CBytes([]byte) unsafe.Pointer {}

// C string to Go string
func C.GoString(*C.char) string {}

// C data with explicit length to Go string
func C.GoStringN(*C.char, C.int) string {}

// C data with explicit length to Go []byte
func C.GoBytes(unsafe.Pointer, C.int) []byte {}
```

**就想减少那一次拷贝**

假设我们可以确定在 c 模块中不会修改 Go 传入的内存，并且 c 函数调用结束之后，c 模块不会再持有这块内存，我们出于性能考虑，想避免这种拷贝，可以这样做：

```
package main

// #include <stdio.h>
// #include <stdlib.h>
//
// static void myprint(char* s, int len) {
//   int i = 0;
//   for (; i < len; i++) {
//     printf("%c", *(s+i));
//   }
//   printf("\n");
// }
import "C"
import "unsafe"

type StringStruct struct {
  str unsafe.Pointer
  len int
}

func main() {
  str := "Hello from stdio"
  ss := (*StringStruct)(unsafe.Pointer(&str))
  c := (*C.char)(unsafe.Pointer(ss.str))
  C.myprint(c, C.int(len(str)))
}
```

这里多说两句，我对于 cgo 在 buffer 传递时，默认使用拷贝方式的理解。

首先，Go 中 string 是 immutable 语义的，我们无法确保 c 模块中不会对传入内存进行修改。相关的内容还可以看看这篇 **[Go 语言中]byte 和 string 类型相互转换时的性能分析和优化。**

更为重要的是，Go 自身的堆内存管理使用了垃圾回收器。那么和 c 语言模块进行交互时，Go 将内存传入 c 模块后，Go 无法知道 c 模块会持有这块内存多久（有可能函数调用结束后，c 模块依然持有这块内存），所以 Go 的解决方案是对内存进行拷贝后传入。一般来说，c 语言里都是秉承哪个模块申请就由哪个模块释放的原则，因为跨库申请释放可能由于各自链接的内存管理库不一致导致出现难以排查的 bug。并且换个角度来说，被调用模块也无法知道传入的内存是在堆上申请还是栈上申请的，是否需要释放。所以 Go 传入 c 模块的内存，c 模块也许会对这块内存再次进行拷贝，但是 c 模块肯定不会释放（即free）传入的这份内存。所以，一般来说，Go 在调用完 c 函数之后，Go 需要释放拷贝生成的这块内存。

很多时候，拷贝体现的是一种解耦的思想，用性能消耗、内存占用量换取可读性和可维护性。

**使用 cgo 时如何定位内存泄漏问题**

以下我们故意制造几种内存泄漏的场景，看 valgrind 和Go pprof是否能检查出来。

测试一，我们注释掉本文第一个例子中的free，使用 valgrind 跑内存泄漏检查，输出信息如下：

```
$valgrind --leak-check=full ./demo

==31055== 17 bytes in 1 blocks are definitely lost in loss record 1 of 4
==31055==    at 0x4C29BC3: malloc (vg_replace_malloc.c:299)
==31055==    by 0x737433: _cgo_d0ada72ffd0d_Cfunc__Cmalloc (_cgo_export.c:30)
==31055==    by 0x45C9DF: runtime.asmcgocall (/usr/local/go/src/runtime/asm_amd64.s:635)
==31055==    by 0xD1A6BF: ???
==31055==    by 0x1FFF00030F: ???
==31055==    by 0x45A057: runtime.goready.func1 (/usr/local/go/src/runtime/proc.go:312)
==31055==    by 0x45B205: runtime.systemstack (/usr/local/go/src/runtime/asm_amd64.s:351)
==31055==    by 0x4331BF: ??? (/usr/local/go/src/runtime/proc.go:1082)
==31055==    by 0x45B098: runtime.rt0_go (/usr/local/go/src/runtime/asm_amd64.s:201)
```

可以看到，虽然 valgrind 给出了definitely lost的结果，但是几乎无法直接找到泄漏的位置。

测试二，我们再来测试在 c 代码中制造泄漏，看 valgrind 是否能查到，测试代码如下：

```
package main

// #include <stdio.h>
// #include <stdlib.h>
// #include <string.h>
//
// static void myprint(char* s) {
//   printf("%s\n", s);
// }
//
// static void f1() {
//   void *p = malloc(128 * 1024 * 1024); // 这里故意申请不释放
//   memset(p, '0', 128 * 1024 * 1024);
// }
import "C"
//import "unsafe"
import _ "net/http/pprof"

func main() {
  cs := C.CString("Hello from stdio")
  C.myprint(cs)
  C.f1()
  //C.free(unsafe.Pointer(cs))
}
```

输出如下：

```
==31701== 134,217,728 bytes in 1 blocks are possibly lost in loss record 5 of 5
==31701==    at 0x4C29BC3: malloc (vg_replace_malloc.c:299)
==31701==    by 0x73754D: f1 (main.go:12)
==31701==    by 0x73754D: _cgo_3679cecbf840_Cfunc_f1 (cgo-gcc-prolog:48)
==31701==    by 0x45CA5F: runtime.asmcgocall (/usr/local/go/src/runtime/asm_amd64.s:635)
==31701==    by 0xD1A6BF: ???
==31701==    by 0x1FFF00031F: ???
==31701==    by 0x45A0D7: runtime.goready.func1 (/usr/local/go/src/runtime/proc.go:312)
==31701==    by 0x45B285: runtime.systemstack (/usr/local/go/src/runtime/asm_amd64.s:351)
==31701==    by 0x43323F: ??? (/usr/local/go/src/runtime/proc.go:1082)
==31701==    by 0x45B118: runtime.rt0_go (/usr/local/go/src/runtime/asm_amd64.s:201)
```

可以看到，valgrind 给出了possibly lost的结果，并且有具体的函数和行号。说明 valgrind 在这种情况可以起作用。

测试三，申请的代码稍微复杂一点，在 c 代码中创建一个新的线程，在线程中制造内存泄漏，代码如下：

```
package main

// #include <stdio.h>
// #include <stdlib.h>
// #include <string.h>
// #include <pthread.h>
//
// static void myprint(char* s) {
//   printf("%s\n", s);
// }
//
// static void *f1(void *q) {
//   void *p = malloc(128 * 1024 * 1024);
//   memset(p, '0', 128 * 1024 * 1024);
//   return NULL;
// }
//
// static void f2() {
//   pthread_t t;
//   pthread_create(&t, NULL, f1, NULL);
// }
import "C"
//import "unsafe"

func main() {
  cs := C.CString("Hello from stdio")
  C.myprint(cs)
  C.f2()
  //C.free(unsafe.Pointer(cs))
}
```

valgrind输出如下：

```
==31858== 134,217,728 bytes in 1 blocks are possibly lost in loss record 6 of 6
==31858==    at 0x4C29BC3: malloc (vg_replace_malloc.c:299)
==31858==    by 0x45197D: f1 (main.go:13)
==31858==    by 0x4E3DDD4: start_thread (in /usr/lib64/libpthread-2.17.so)
==31858==    by 0x514FEAC: clone (in /usr/lib64/libc-2.17.so)
```

可以看到，这种情况 valgrind 也可以检查出来。

测试四，用go pprof分析，分析方法见 Go 语言 pprof 备忘录，代码如下：

```
package main

// #include <stdio.h>
// #include <stdlib.h>
// #include <string.h>
//
// static void myprint(char* s) {
//   printf("%s\n", s);
// }
//
// static void f1() {
//   void *p = malloc(128 * 1024 * 1024);
//   memset(p, '0', 128 * 1024 * 1024);
// }
import "C"
//import "unsafe"
import _ "net/http/pprof"
import "net/http"

func main() {
  cs := C.CString("Hello from stdio")
  C.myprint(cs)
  C.f1()
  //C.free(unsafe.Pointer(cs))
  http.ListenAndServe("0.0.0.0:10001", nil)
}
```

go pprof的结果是，它并不记录C.CString申请的内存，也不记录 c 代码中申请的内存。

**其他**

我个人猜测go pprof只监测通过 Go 垃圾回收器申请和释放的内存，C.CString以及 c 代码中的内存申请都没有经过 gc，所以无法监测。

文档中也有相应的描述，如下：

```
As a special case, C.malloc does not call the C library malloc directly but instead calls a Go helper function that wraps the C library malloc but guarantees never to return nil. If C's malloc indicates out of memory, the helper function crashes the program, like when Go itself runs out of memory. Because C.malloc cannot fail, it has no two-result form that returns errno.
```

转自：

github.com/q191201771