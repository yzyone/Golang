首先编写一个rc文件，如main.rc，内容如下：

    IDI_ICON1 ICON "icon.ico"
    1 24 "main.exe.manifest"

icon指的是程序的图标，下边的manifest是让程序使用windows主题,string table 、version等按照普通rc文件写入即可。

使用windres将rc文件编译为syso文件，go语言在最新版本中已经支持syso文件的链接并且会搜索当前目录，自动链接：

    windres -o main-res.syso main.rc
    go build -ldflags '-H windowsgui -w'

这样生成的exe就有了图标，并且应用了windows的主题。

其中ldflags中的参数，"-H windowsgui"为隐藏命令行窗口，因为我写的是gui程序，"-w"是裁剪gdb调试信息，这样生成的exe体积会小一些。

用到的windres提供下载，也可以从MinGW中提取：

    windres.zip

用到的文件提供下载，包括icon.ico 、 manifest 及rc文件：

    go_res.zip