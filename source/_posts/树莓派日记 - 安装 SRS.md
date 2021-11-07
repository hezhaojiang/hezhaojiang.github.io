---
title: 树莓派日记 - 安装 SRS
categories:
  - 树莓派日记
tag:
    - SRS
abbrlink: f3c9706d
date: 2020-10-17 07:16:50
---
`SRS/3.0`，是一个流媒体集群，支持 `RTMP`/`HLS`/`WebRTC`/`SRT`/`GB28181`，高效、稳定、易用，简单而快乐。

<!--more-->

## SRS 介绍

`SRS` 定位是运营级的互联网直播服务器集群，追求更好的概念完整性和最简单实现的代码。`SRS` 提供了丰富的接入方案将 `RTMP` 流接入 `SRS`， 包括推送 `RTMP` 到 `SRS`、推送 `RTSP`/`UDP`/`FLV` 到 `SRS`、拉取流到 `SRS`。`SRS` 还支持将接入的 `RTMP` 流进行各种变换，譬如将 `RTMP` 流转码、流截图、 转发给其他服务器、转封装成 `HTTP-FLV` 流、转封装成 HLS、 转封装成 `HDS`、转封装成 `DASH`、录制成 `FLV`/`MP4`。`SRS` 包含支大规模集群如 `CDN` 业务的关键特性， 譬如 `RTMP` 多级集群、源站集群、`VHOST` 虚拟服务器 、 无中断服务 `Reload`、`HTTP-FLV` 集群。此外，`SRS` 还提供丰富的应用接口， 包括 `HTTP` 回调、安全策略 `Security`、`HTTP` `API` 接口、 `RTMP` 测速。`SRS` 在源站和 `CDN` 集群中都得到了广泛的应用 `Applications`。

## 依赖环境

``` bash
## python2
$ sudo apt-get install python python-dev
## net-tools
$ sudo apt-get install net-tools
## cmake
$ sudo apt-get install -y cmake
## gcc
$ sudo apt-get install build-essential
```

## 安装过程

### Get SRS

``` bash
# git clone https://gitee.com/winlinvip/srs.oschina.git srs
# cd srs/trunk && git remote set-url origin https://github.com/ossrs/srs.git && git pull
$ git clone https://github.com/ossrs/srs.git && cd srs/trunk
```

### Build SRS

``` bash
$ ./configure && make
```

### 安装过程中遇到的问题

#### Build openssl-1.1.0e failed, ret=2

`Ubuntu 20.04 LTS` 中 `openssl` 存在缺陷，无法编译通过，建议使用以下配置

``` bash
$ ./configure --full --use-sys-ssl --without-utest
```

## 配置

### 添加系统服务

``` bash
$ sudo ln -sf /usr/local/srs/etc/init.d/srs /etc/init.d/srs
$ sudo cp -f /usr/local/srs/usr/lib/systemd/system/srs.service /usr/lib/systemd/system/srs.service
$ sudo systemctl daemon-reload
```

此时可用以下命令启动 `SRS` 服务：

``` bash
$ sudo systemctl start srs
```

### 设置开机自启

``` bash
$ sudo systemctl enable srs
```

### 启用 HLS 和 HTTP_FLV

修改 `SRS` 的配置文件 `/usr/local/srs/conf/srs.conf`：

``` bash
 # main config for srs.
 # @see full.conf for detail config.

 listen              1935;
 max_connections     1000;
 srs_log_tank        file;
 srs_log_file        ./objs/srs.log;
 http_api {
     enabled         on;
     listen          1985;
 }
 http_server {
     enabled         on;
     listen          8080;
     dir             ./objs/nginx/html;
 }
 stats {
     network         0;
     disk            sda sdb xvda xvdb;
 }
 vhost __defaultVhost__ {
     # HTTP_FLV Configure
     http_remux{
         enabled    on;
         mount      [vhost]/[app]/[stream].flv;
         hstrs      on;
     }
     # HLS Configure
     hls{
         enabled       on;
         hls_path      ./objs/nginx/html;
         hls_fragment  10;
         hls_window    60;
     }
 }
```

重新启动 `SRS` 服务：`sudo systemctl restart srs`。

## 测试

### 使用 ffmpeg 推流

在服务器上使用 ffmpeg 向 SRS 服务进行推流，例如：

```bash
ffmpeg -i "rtsp://username:password@192.168.199.242/ch1/main/av_stream" \
    -vcodec libx264 -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1 \
    -f flv rtmp://localhost:1935/hls/movie
```

### 使用 VLC 拉流

使用可以连接到服务器的计算机打开 `VLC` 播放器进行拉流，`VLC` 的 `URL` 格式如下：

``` node
RTMP : rtmp://192.168.199.152:1935/hls/movie
RTMP : http://192.168.199.152:8080/hls/movie.m3u8
HTTP_FLV : http://192.168.199.152:8080/hls/movie.flv
```

拉流播放效果如图所示：

![HTTP_FLV](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201122220304.png)

## 参考资料

* [1] [ossrs/srs: SRS is a RTMP/HLS/WebRTC/SRT/GB28181 streaming cluster, high efficiency, stable and simple.](https://github.com/ossrs/srs)
