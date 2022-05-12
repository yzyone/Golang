# GO语言生成exe加图标 #

第一步需要下载一个第三方库

	go get github.com/akavel/rsrc

下载完成之后来到你设置GOPATH环境变量的目录

环境变量\src\github.com\akavel\rsrc 然后编译一下rsrc.go编译成exe可执行文件

拷贝rsrc.exe到你的GOPATH目录

 

创建manifest文件, 命名：main.exe.manifest 

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
<assemblyIdentity
    version="1.0.0.0"
    processorArchitecture="x86"
    name="controls"
    type="win32"
></assemblyIdentity>
<dependency>
    <dependentAssembly>
        <assemblyIdentity
            type="win32"
            name="Microsoft.Windows.Common-Controls"
            version="6.0.0.0"
            processorArchitecture="*"
            publicKeyToken="6595b64144ccf1df"
            language="*"
        ></assemblyIdentity>
    </dependentAssembly>
</dependency>
</assembly>
```

命令行：

	rsrc.exe -manifest main.exe.manifest -ico 图标的名字.ico -o main.syso
	go build -o main.exe

CMD运行这俩条命令就可以了

亲测(注意目录别搞混了就可以)

————————————————

版权声明：本文为CSDN博主「博士TEL」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/qq_23257367/article/details/117558470