# comcast：基于 go 的网络环境模拟工具

分布式系统调试时模拟极端网络环境（高延迟、高丢包）的情况非常重要，github 上有个开源方案：[comcast（7.2k star）](https://github.com/tylertreat/comcast) 可以轻易实现网络环境模拟，

直接通过命令行设置服务器网络环境（10% 丢包率）：

```
$ comcast --device=eth0 --latency=250 --target-bw=1000 --packet-loss=10%
```

也可以选择已有的配置，代码也比较简单，可以直接读源码学一下实现原理。

| Name            | Latency | Bandwidth | Packet-loss |
| --------------- | ------- | --------- | ----------- |
| GPRS (good)     | 500     | 50        | 2           |
| EDGE (good)     | 300     | 250       | 1.5         |
| 3G/HSDPA (good) | 250     | 750       | 1.5         |
| DIAL-UP (good)  | 185     | 40        | 2           |
| DSL (poor)      | 70      | 2000      | 2           |
| DSL (good)      | 40      | 8000      | 0.5         |
| WIFI (good)     | 40      | 30000     | 0.2         |
| Starlink        | 20      | -         | 2.5         |

[golang](https://hackertalk.net/tags/golang)
