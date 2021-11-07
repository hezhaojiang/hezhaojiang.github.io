---
title: NRM - 环境搭建
tags:
  - NRM
abbrlink: 71c958c4
date: 2020-10-07 11:15:48
---
NRM(Nginx-rtmp-module) 的出现，使得很多非专业的流媒体开发工程师也可以简单、迅速地搭建流媒体服务器。

在 NRM 安装之前，首先要确保完成了 {% post_link "Nginx - 环境搭建" %} 中的内容，然后再进行后续操作。

<!--more-->

## NRM 介绍

## 安装前的准备

### 下载 NRM

可以在 Github 中的 nginx-rtmp-module 项目仓库：<https://github.com/arut/nginx-rtmp-module/releases> 获取 nginx-rtmp-module 源码包。

至本文编写时，最新的 nginx-rtmp-module 稳定版本为 `nginx-rtmp-module-1.2.1`，下文也是用该版本的源码包 `nginx-rtmp-module-1.2.1.tar.gz` 进行安装。

将下载好的源码包 `nginx-rtmp-module-1.2.1.tar.gz` 放到准备好的 `Nginx` 源码目录（比如我的目录为 `/root/nginx/`）中，通过命令解压：

``` bash
# git clone https://github.com/arut/nginx-rtmp-module
$ tar xzvf nginx-rtmp-module-1.2.1.tar.gz
$ mv nginx-rtmp-module-1.2.1 rtmp-module
```

## 安装

进入 `Nginx` 源码目录（比如我的目录为 `/root/nginx/nginx-1.18.0/`），执行以下命令：

``` bash
$ ./configure --add-module=/path/src/rtmp-module/
# My configure option
$ ./configure --add-module=/root/nginx/rtmp-module/ --prefix=/usr/local/nginx --with-http_ssl_module --pid-path=/var/run/nginx.pid --with-debug
$ make && sudo make install
```

## 配置

### 修改配置文件

1. 从原配置文件中拷贝一份作为新配置文件的模板：

    ``` bash
    $ cd /usr/local/nginx/conf/
    $ cp nginx.conf live.conf
    ```

2. 在新配置文件末尾加入以下内容：

    ``` linux
    rtmp {
        server {
            listen 1935;
            application mylive {
                live on;
            }
        }
    }
    ```

3. 启用新配置文件：

    ==1. Systemctl 工具配置（推荐）==

    - 修改系统服务脚本（`sudo vim /lib/systemd/system/nginx.service`）：

        ``` linux
        # 找到如下行
        ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
        # 修改为
        ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/live.conf
        ```

    - 重启 Nginx 服务使新配置生效：

        ``` bash
        [root@VM_0_6_centos conf]# sudo systemctl restart nginx
        ```

    ==Service 工具配置==

    - 修改系统服务脚本（`vim /etc/init.d/nginx`）：

        ``` linux
        # 找到如下行
        NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"
        # 修改为
        NGINX_CONF_FILE="/usr/local/nginx/conf/live.conf"
        ```

    - 检查配置文件正确性：

        ``` bash
        [root@VM_0_6_centos conf]# service nginx configtest
        nginx: the configuration file /usr/local/nginx/conf/live.conf syntax is ok
        nginx: configuration file /usr/local/nginx/conf/live.conf test is successful
        ```

    - 重启 Nginx 服务使新配置生效：

        ``` bash
        [root@VM_0_6_centos conf]# service nginx restart
        Restarting nginx (via systemctl):                          [  OK  ]
        ```

## 测试

### 使用 ffmpeg 推流

在服务器上使用 ffmpeg 向 Nginx-rtmp-module 服务进行推流：

```bash
ffmpeg -re -i xxx.flv -r 25 -b 1M -f flv rtmp://127.0.0.1:1935/url
# 示例：
ffmpeg -re -i Read_Book_852x480.flv -ar 22050 -f flv rtmp://127.0.0.1:1935/mylive/1
ffmpeg -re -stream_loop -1 -i Read_Book_1280x720.flv -vcodec libx264 -f flv rtmp://127.0.0.1:1935/mylive/1
```

### 使用 VLC 拉流

使用可以连接到服务器的计算机打开 `VLC` 播放器进行拉流：

![VLC 拉流测试](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201007225859.png)

## 参考资料

* [1] 中国工信出版集团 《直播系统开发 基于 Nginx 与 Nginx-rtmp-module》
