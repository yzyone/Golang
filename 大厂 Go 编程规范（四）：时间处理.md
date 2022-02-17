# 大厂 Go 编程规范（四）：时间处理 #

## 1、time ##

go time 是基于 int 所以可以通过直接对比int 大小确定时间早晚

```
func isActive(now, start, stop int) bool {
  return start <= now && now < stop
}
```

但时间的对比，最好使用 time，如下

```
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

可读性更好。另外，encoding/json 通过其 UnmarshalJSON method 方法支持将 time.Time 编码为 RFC 3339 字符串。

## 2、Duration ##

时间段处理也是类似，下面的代码poll 方法传入 10

```
func poll(delay int) {
  for {
    // ...
    time.Sleep(time.Duration(delay) * time.Millisecond)
  }
}
poll(10) 
```

但谁能知道传入的 10代表的是 10s 和10ms ，所以更推荐的做法就是直接传入Duration

```
func poll(delay time.Duration) {
  for {
    // ...
    time.Sleep(delay)
  }
}
poll(10*time.Second)
```

这样方法调用者，就可以根据自己的需求传入对应的时间段。而且 flag 通过 time.ParseDuration 已经支持 time.Duration类型。

最后，如果外部系统不支持time 类型的时候，比如需要将 duration json 的时候，这种命名方式让使用者很难了解interval 的单位。

```
type Config struct {
  Interval int `json:"interval"`
}
```

所以更加推荐这种写法

```
// {"intervalMillis": 2000}
type Config struct {
  IntervalMillis int `json:"intervalMillis"`
}
```

这样调用者就很清晰地了解到单位是毫秒了。