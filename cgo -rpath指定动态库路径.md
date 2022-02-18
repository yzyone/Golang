# cgo -rpath指定动态库路径 #

```
// #cgo CFLAGS: -Wall
// #cgo LDFLAGS: -Wl,-rpath="/home/liuliang/ffmpeg-build/lib"
// #cgo LDFLAGS: -L/home/liuliang/workspace/wetrip_ffmpeg_demuxer/Debug
// #cgo LDFLAGS: -L/home/liuliang/workspace/wetrip_ffmpeg_demuxer
// #cgo LDFLAGS: -lwetrip_ffmpeg_demuxer -lstdc++ -ljpeg  -lpthread -lrt
// #cgo LDFLAGS: -L/home/liuliang/ffmpeg-build/lib
// #cgo  LDFLAGS: -lavformat -lavcodec -lswscale -lavutil -lswresample
// #include "test.h"
// #include <stdio.h>
// #include <stdlib.h>
// #include "wetrip_ffmpeg_demuxer.h"
```
