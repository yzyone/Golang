# golang之cgo---调用C/C++动态库函数 #

之前说过golang调用C代码的方式可以通过cgo或者是swig，而cgo是不能使用C++相关的东西的，比如标准库或者C++的面向对象特性。怎么办，将c++的功能函数封装成C接口，然后编译成动态库，或者是功能较为简单的可以直接嵌入到go源文件中。

cgo的使用是在linux平台上，在windows平台上可以配置交叉编译器。

动态库头文件：myfuns.h

```
#pragma once

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdbool.h>

void fun1();

void fun2(int a);

int fun3(void **b);

// others
```

动态库名：myfuns.so

项目简化结构：

```
|-project
|  |-lib
|  |  |-myfuns.so
|  |-include
|  |  |-myfuns.h
|  |-src
|  |  |-main.go
|  |-pkg
|  |-bin
```

go链接动态库：main.go

```
package main

/*
#cgo CFLAGS : -I../include
#cgo LDFLAGS: -L../lib -lmyfuns

#include "myfuns.h"
*/
import "C"

import (
    "fmt"
)

func main() {
    // 调用动态库函数fun1
    C.fun1()
    // 调用动态库函数fun2
    C.fun2(C.int(4))
    // 调用动态库函数fun3
    var pointer unsafe.Pointer
    ret := C.fun3(&pointer)
    fmt.Println(int(ret))
}
```

通过CFLAGS配置编译选项，通过LDFLAGS来链接指定目录下的动态库。这里需要注意的一个地方就是import "C"是紧挨着注释的，没有空行。

————————————————

版权声明：本文为CSDN博主「别打名名」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/FreeApe/article/details/51927615