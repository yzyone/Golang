# Golang 字符串（string）与字节数组([]byte)一行代码互转 #

文章目录

- 一、字符串与字节数组？
- 二、详细代码
- 1.简单的方式字节转字符串
- 2.简单的字符串转字节数组
- 3.字节转字符串
- 4.字符串转字节数组
- 5.完整运行测试
- 总结

## 一、字符串与字节数组？ ##

字符串是 Go 语言中最常用的基础数据类型之一，本质上是只读的字符型数组，虽然字符串往往都被看做是一个整体，但是实际上字符串是一片连续的内存空间。

Go 语言中另外一个类型字节（Byte）。在ASCII中，一个英文字母占一个字节的空间，一个中文汉字占两个字节的空间。英文标点占一个字节，中文标点占两个字节。一个Byte数组中的元素对应一个ASCII码。

## 二、详细代码 ##

**1.简单的方式字节转字符串**

代码如下（示例）：

```
func Bytes2String(data []byte) string {
	return string(data)
}
```

**2.简单的字符串转字节数组**

代码如下（示例）：

```
func String2Bytes(data string) []byte {
	return []byte(data)
}
```

ps:以上两种简单的方式略过不提，主要实验 unsafe 正常转译

**3.字节转字符串**

代码如下（示例）：

```
func BytesToString(data []byte) string {
	return *(*string)(unsafe.Pointer(&data))
}
```

**4.字符串转字节数组**

代码如下（示例）：

```
func StringToBytes(data string) []byte {
	return *(*[]byte)(unsafe.Pointer(&data))
}
```

**5.完整运行测试**

代码如下（示例）：

```
func BytesToString(data []byte) string {
	return *(*string)(unsafe.Pointer(&data))
}


func StringToBytes(data string) []byte {
	return *(*[]byte)(unsafe.Pointer(&data))
}

func main() {
	str := "hello world!"

	fmt.Println(str)

	a := StringToBytes(str)

	fmt.Println(a)

	b := BytesToString(a)

	fmt.Println(b)
}
```

**结果（示例）：**

```
hello world!
[104 101 108 108 111 32 119 111 114 108 100 33]
hello world!     
```

成功转译出Hello world!

总结
两个方法来记住字节数组与字符串互转，简单直接，实用性拉满。

希望这个博客能对你有所益处。我是轻王，我为自己代言。

————————————————

版权声明：本文为CSDN博主「猫轻王」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/moer0/article/details/122934188