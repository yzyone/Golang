
# Golang和xterm.js实现webssh #

**Github地址：https://github.com/widaT/webssh**

**WEBSSH**

基于vue、xterm、golang实现的web ssh客户端程序，支持录像回看

**特性**

前后端分离，前端使用xterm、vue，后端使用golang写的服务
支持录像审计，支持录像回看

**run demo**

编译前端程序

```
$ cd front
$ npm -i
$ npm run build # 可以看到在front生成一个dist目录，里头就是编译后的前端文件
```

编译golang程序
修改main.go文件中目标主机和登录方式

```
confing := &webssh.WebSSHConfig{
		Record:     true,
		RecPath:    "./rec/cast/",
		RemoteAddr: "localhost:22",
		User:       "wida",
		Password:   "wida",
		AuthModel:  webssh.PASSWORD,
	}
$ go build -o webssh main.go
$ ./webssh
```

用浏览器打开http://localhost:8080/#/term

**查看录像**

用浏览器打开http://localhost:8080/#/rec，顶部有选择器，选择生成的文件播放（手动点击播放）。

**动画演示**

![](./webssh/bc5f63c6ae66422380dbc8337c7e3587.gif)

golang和xterm.js实现webssh
