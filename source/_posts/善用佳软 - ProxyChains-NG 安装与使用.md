---
title: 善用佳软 - ProxyChains-NG 安装与使用
categories:
  - 工具软件
tags:
  - Proxy
abbrlink: b7dd7417
date: 2020-10-07 14:41:29
---
因为某些原因，我们需要在命令行下载一些国外的资源，这个时候如果使用 `wget`，`curl`，或者 `aria2c` 的时候，往往又没有速度。这个时候我们需要使用代理来进行加速。

我本地搭的有 `ss`，但 `ss` 只支持 `socks5` 协议，而 `wget`，`curl` 之类使用 `http_proxy` 进行代理的软件往往无法通过代理进行科学上网。我们可以利用一款名叫 `ProxyChains-NG` 的软件，`chains` 故名思义，可以支持代理链，这样我们可以在内部使用 `Proxychains` 把 `http_proxy` 代理到 `socks5` 上，达到想要的效果。

> 项目开源地址：Github : <https://github.com/rofl0r/proxychains-ng>

<!--more-->

## 安装

``` bash
# git clone https://gitee.com/hezhaojiang/proxychains-ng
# cd proxychains-ng/ && git remote set-url origin https://github.com/rofl0r/proxychains-ng && git pull
$ git clone https://github.com/rofl0r/proxychains-ng && cd proxychains-ng/
./configure && make
# 以下命令非 root 用户注意使用 sudo 命令
make install
make install-config
```

## 配置

编辑 `/etc/proxychains.conf`, 在最后一行改为:

``` linux
socks5 192.168.199.135 10808
```

其中以下参数需要根据实际情况自行配置：

- `socks5` 是网络代理协议
- `192.168.199.135` 是代理服务器地址
- `10808` 是代理服务器监听端口

## 使用

不加代理明显会超时, 加代理后会提示:

``` bash
ubuntu@ubuntu:~$ proxychains4 wget https://www.google.com
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/libproxychains4.so
[proxychains] DLL init: proxychains-ng 4.14
--2020-10-10 13:44:49--  https://www.google.com/
Resolving www.google.com (www.google.com)... 224.0.0.1
Connecting to www.google.com (www.google.com)|224.0.0.1|:443... [proxychains] Strict chain  ...  192.168.199.135:10808  ...  www.google.com:443  ...  OK
connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘index.html’

index.html     [ <=>                                      ]  11.79K  --.-KB/s    in 0.1s

2020-10-10 13:44:50 (83.6 KB/s) - ‘index.html’ saved [12068]
```
