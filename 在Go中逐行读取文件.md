## 在Go中逐行读取文件

在Go 1.1和更新版本中，最简单的方法是使用[`bufio.Scanner`](http://golang.org/pkg/bufio/#Scanner)。下面是一个从文件中读取行的简单示例：

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
)

func main() {
    file, err := os.Open("/path/to/file.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    // optionally, resize scanner's capacity for lines over 64K, see next example
    for scanner.Scan() {
        fmt.Println(scanner.Text())
    }

    if err := scanner.Err(); err != nil {
        log.Fatal(err)
    }
}
```

这是从`Reader`逐行读取数据的最简洁的方法。

有一点需要注意:扫描程序将在超过65536个字符的行中出错。如果您知道您的线路长度大于64K，请使用`Buffer()`方法增加扫描仪的容量：

```go
...
scanner := bufio.NewScanner(file)

const maxCapacity = longLineLen  // your required line length
buf := make([]byte, maxCapacity)
scanner.Buffer(buf, maxCapacity)

for scanner.Scan() {
...
```