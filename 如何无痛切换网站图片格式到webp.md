# 如何无痛切换网站图片格式到webp



[M酷](https://bbchin.com/s/about "M酷")

2021-09-07/6 评论/12 点赞/1,571 阅读/3,717 字/推送成功！

09/07

![优雅的让 Halo 支持 webp 图片输出](https://bbchin.com/upload/2021/10/webp_compare-8a9a115ddb094290b53e4d21201254d4.webp)

## 是什么？

---

> WebP的有损压缩算法是基于VP8视频格式的帧内编码[17]，并以RIFF作为容器格式。[2] 因此，它是一个具有八位色彩深度和以1:2的比例进行色度子采样的亮度-色度模型（YCbCr 4:2:0）的基于块的转换方案。[18] 不含内容的情况下，RIFF容器要求只需20字节的开销，依然能保存额外的 元数据(metadata)。[2] WebP图像的边长限制为16383像素。

在 `WebP` 的官网中，我们可以发现 Google 是这样宣传 `WebP` 的：

> WebP lossless images are 26% smaller in size compared to PNGs. WebP lossy images are 25-34% smaller than comparable JPEG images at equivalent SSIM quality index.

简单来说，`WebP` 图片格式的存在，让我们在 `WebP` 上展示的图片体积可以有较大幅度的缩小，也就带来了加载性能的提升。（摘自 [让站点图片加载速度更快——引入 WebP Server 无缝转换图片为 WebP](https://nova.moe/re-introduce-webp-server))

![pic_compare.png](https://bbchin.com/upload/2021/09/pic_compare-b12b1ac24136425799a09709b32f581b.png)

## 怎么做？

---

那么如何优雅的在不替换图片地址的情况下，将图片转为 `webp` 格式然后输出呢？

这时候就可以使用 [webp-sh](https://github.com/webp-sh) 组织最新开源的 [webp_server_go](https://github.com/webp-sh/webp_server_go) 工具了。

**原理：**

当我们请求一张图片的时候使用 web 代理工具转发到 `webp_server_go` 应用进行处理，处理完成之后返回 webp 格式的图片，并且会保留处理后的图片以供后面的访问。

目前大部分主流浏览器都已经支持了 `webp` 图片的显示，除了 `Safari`，但是不必担心，`webp_server_go` 会自动判断请求来源是否为 `Safari`，如果是，那么会返回原图。

下面将提供两种 web 服务器的代理方法。

> 此教程以 `CentOS 7.x` 为例，其他发行版本大同小异。另外，此教程只针对于 `Halo`，其他 web 程序可能在 `config.json` 部分有所不同，建议参考仓库的 `README`。

### 部署 webp_server_go

在 [仓库](https://github.com/webp-sh/webp_server_go) 的 README 中已经大致讲解了部署方法，这里针对 [Halo博客平台](https://halo.run/) 详细说明一下。

#### 1、下载官方编译好的 `webp_server_go` 二进制文件

> 如果你有能力，也可以自行编译。

① 新建一个存放二进制文件和 `config.json` 文件的目录（可自定义）：

`mkdir /opt/webps
cd /opt/webps` 

② 下载二进制文件（最新版本请访问 [releases](https://github.com/webp-sh/webp_server_go/releases)）：

`wget https://github.com/webp-sh/webp_server_go/releases/download/0.3.2/webp-server-linux-amd64 -O webp-server` 

③ 给予目录执行权限：

`chmod +x webp-server`

#### 2、创建 `config.json`

`{
    "HOST": "127.0.0.1",
    "PORT": "3333",
    "QUALITY": "80",
    "MAX_JOB_COUNT": "10",
    "IMG_PATH": "/root/.halo",
    "EXHAUST_PATH": "/root/.halo/cache",
    "ALLOWED_TYPES": ["jpg", "png", "jpeg", "bmp"]
}` 

**参数解释：**

`*   HOST：一般不修改

* PORT：webp_server_go 的运行端口
* QUALITY：转换质量，默认为 80%
* MAX_JOB_COUNT：同一时间最大的转换数，超过会排队
* IMG_PATH：固定格式，/运行 Halo 的用户名/.halo
* EXHAUST_PATH：固定格式，/运行 Halo 的用户名/.halo/cache
* ALLOWED_TYPES：需要转换的格式` 

#### 3、使用 `systemd` 进行状态管理

① 创建 `service` 文件：

`vim /etc/systemd/system/webps.service`

② 写入配置：

`[Unit]
Description=WebP Server
Documentation=https://github.com/n0vad3v/webp_server_go
After=nginx.target

[Service]
Type=simple
StandardError=journal
AmbientCapabilities=CAP_NET_BIND_SERVICE
WorkingDirectory=/opt/webps
ExecStart=/opt/webps/webp-server --config /opt/webps/config.json
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=3s

[Install]
WantedBy=multi-user.target` 

> 需要注意的是，`ExecStart` 命令中的程序路径和配置文件路径一定要正确，结合你的实际情况填写。

③ 然后执行：

`systemctl daemon-reload
systemctl enable webps.service
systemctl start webps.service` 

④ 查看运行状态：

`systemctl status webps.service`

如果没有问题，那么会输出以下日志：

`WebP Server is running at 127.0.0.1:3333`

### 使用 `Nginx` 进行代理

> 如果你的 `Halo` 是使用 `Nginx` 反向代理的话。

#### 1、修改 `halo.conf` 或自己的 `nginx.conf` 文件

在 `server` 节点添加：

`location ^~ /upload/ {
  proxy_pass http://127.0.0.1:3333;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_hide_header X-Powered-By;
  proxy_set_header HOST $http_host;
  add_header Cache-Control 'no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0';
}` 

#### 2、重载 Nginx 配置：

`# 检查配置文件是否有问题
nginx -t 

# 重启nginx

nginx -s reload` 

### 使用 `Caddy` 进行代理

> 如果你的 `Halo` 是使用 `Caddy` 反向代理的话。

#### 1、修改 `Caddyfile`

在你域名节点下添加：

`proxy /upload/ localhost:3333 {
  transparent
}` 

#### 2、重启 `Caddy`：

`service caddy restart`

教程完毕，下面讲一下如何验证是否生效。

### 验证是否生效

![webp_img](https://bbchin.com/upload/2021/10/webp_img-be5807d3c5834876b43a554d247a1cd6.png)

注意看 `类型(Type)` 列，图片的返回格式已经变为 `webp`，而且图片大小已经远远降低，此时说明你的配置已经成功了。😁 have fun!

如果用的开心，请关注一下 [GitHub - webp-sh/webp_server_go: Go version of WebP Server. A tool that will serve your JPG/PNGs as WebP format with compression, on-the-fly.](https://github.com/webp-sh/webp_server_go) 哦！另外，他们还有其他语言的版本，请查看 [WebP Server · GitHub](https://github.com/webp-sh)。

### 重启、停止和关闭服务

`// 查看 webps.server 状态
systemctl status webps.service

// 重启 webps.service
systemctl restart webps.service

// 停止 webps.service
systemctl stop webps.service

// 关闭 webps.service
systemctl disable webps.service` 

如果后续不想使用 `webp_server_go` 了，先停止并关闭 `webps.service`，然后恢复以前的 `Nginx` 或 `Caddy` 配置文件并重启即可。

## 👍 优点

- 接入简单，可在不替换原图资源的情况下进行转换；
- 对大图片或图片量比较大的网站来说，可以显著提高首屏加载速度，优化网站性能指标；
- 针对不支持的平台，有回退方案；

## 🤔 问题

由于这种方式是通过服务端即时对请求的图片进行转换后返回的，因此带来了如下问题：

- 浏览器无法再复用之前图片的请求缓存来优化性能了；
  - 正常情况下浏览器会通过缓存来提高二次加载性能
- 每次服务端都得转换一次，访问频率高的情况下服务端压力大；
- 部分特殊场景可能导致图片转换失败，最终无法展示。
  - 如在未限制并发转换数的情况下，一个页面同时请求了太多图片（如图片类网站），很可能导致服务器资源消耗殆尽而罄机
  - 单纯通过文件扩展名无法准确判断文件类型，此时如果一张图片已经是 `webp`，只是扩展名被改为 `jpg` ，那么他可能会重复转换甚至失败

## 参考

- [GitHub - webp-sh/webp_server_go: Go version of WebP Server. A tool that will serve your JPG/PNGs as WebP format with compression, on-the-fly.](https://github.com/webp-sh/webp_server_go)
- [让站点图片加载速度更快——引入 WebP Server 无缝转换图片为 WebP](https://nova.moe/re-introduce-webp-server/)
- [个人网站无缝切换图片到 webp](https://www.bennythink.com/flying-webp.html)

[webp](https://bbchin.com/tags/webp)

 版权归属： M酷

 本文链接： [如何无痛切换网站图片格式到webp](https://bbchin.com/archives/towebp)

 许可协议： 本文使用《[署名-非商业性使用-相同方式共享 4.0 国际 (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh)》协议授权
