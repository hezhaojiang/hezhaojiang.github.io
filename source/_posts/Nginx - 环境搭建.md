---
title: Nginx - 环境搭建
tags:
  - Nginx
abbrlink: 8f761f75
date: 2020-10-05 13:48:24
---
## 准备工作

### Linux 操作系统

首先我们需要一个内核为 `Linux 2.6` 及以上版本的操作系统，因为 `Linux 2.6` 及以上内核才支持 `epoll`，而在 `Linux` 上使用 `select` 和 `poll` 来解决事件的多路复用，是无法解决高并发压力问题的。

我们可以用 `uname -a` 命令来查询 `Linux` 内核版本，例如：

``` linux
# Tencent Cloud Server：
[root@VM_0_6_centos ~]# uname -a
Linux VM_0_6_centos 3.10.0-1062.9.1.el7.x86_64 #1 SMP Fri Dec 6 15:49:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

<!-- more -->

> 也推荐使用树莓派来安装运行 `Linux` 操作系统，具体步骤可参考：{% post_link "树莓派日记 - 安装 Ubuntu 20.04" %}

### 安装必要软件

#### gcc 编译器

``` bash
# For Centos
$ sudo yum install -y gcc gcc-c++
# For Ubuntu
$ sudo apt-get install -y build-essential
```

#### pcre 库

``` bash
# For Centos
$ sudo yum install -y pcre pcre-devel
# For Ubuntu
$ sudo apt-get install -y libpcre3 libpcre3-dev
```

#### zlib 库

``` bash
# For Centos
$ sudo yum install -y zlib zlib-devel
# For Ubuntu
$ sudo apt-get install -y zlib1g-dev
```

#### OpenSSL 库

``` bash
# For Centos
$ sudo yum install -y openssl openssl-devel
# For Ubuntu
$ sudo apt-get install -y openssl libssl-dev
```

### 创建部署目录

要使用 `Nginx`，还需要再 `Linux` 文件系统上准备以下目录：

1. `Nginx` 源代码存放目录
    用于放置从官网下载的 `Nginx` 源码文件，以及第三方或我们自己写的模块代码文件
2. `Nginx` 编译阶段产生的中间文件存放目录
    默认会将该目录命名为 obgs，并放置于 `Nginx` 源代码目录下
3. 部署目录
    默认路径为 `/usr/local/nginx`
4. 日志存放目录
    日志文件通常会比较大，尤其是在研究 `Nginx` 底层架构和自己编写模块代码时，需要打开 `debug` 级别的日志，故建议预先分配一个拥有较大磁盘空间的目录用来存放日志。

### 优化内核参数

`Nginx` 提供 `Web` 服务时 `Linux` 内核参数调整是必不可少的。

首先需要修改 `/etc/sysctl.conf` 来更改内核参数，各参数的意义及修改值如下：

### 获取 Nginx 源码

可以在 Nginx 官方网站：<http://nginx.org/en/download.html> 获取 Nginx 源码包

至本文编写时，最新的 `Nginx` 稳定版本为 `nginx-1.18.0`，下文也是用该版本的源码包进行安装

将下载好的源码包 `nginx-1.18.0.tar.gz` 放到准备好的 `Nginx` 源代码目录中，通过命令解压：

``` bash
## sudo apt install -y mercurial
## hg clone http://hg.nginx.org/nginx
$ tar xzvf nginx-1.18.0.tar.gz
```

## 编译 & 安装

### 编译 & 安装命令

安装过程极为容易，依次执行以下命令即可完成安装：

``` bash
$ cd nginx-1.18.0
$ ./configure
# My configure option
$ ./configure--with-http_ssl_module --pid-path=/run/nginx.pid
$ make -j4 && sudo make install
```

### 编译前的可选参数

以上命令中，`configure` 命令可以携带参数，以下是对其部分常用参数的说明：

* `--prefix=` 指定安装目录
* `--sbin-path=` 指定执行程序文件存放位置
* `--modules-path=` 指定第三方模块的存放路径
* `--conf-path=` 指定配置文件存放位置
* `--error-log-path=` 指定错误日志存放位置
* `--http-log-path=` 指定 `access log` 路径
* `--pid-path=` 指定 `pid` 文件存放位置
* `--lock-path=` 指定 `lock` 文件存放位置
* `--user=` 指定程序运行时的非特权用户
* `--group=` 指定程序运行时的非特权用户组
* `--builddir=`  指向编译目录
* `--with-threads` 启用 `thread pool` 支持
* `--with-http_ssl_module` 启用 `https` 支持
* `--with-http_v2_module` 启用 `ngx_http_v2_module` 支持
* `--with-ipv6` 启用 `ipv6` 支持
* `--without-http`  禁用 `http server` 功能
* `--without-http-cache`  禁用 `http cache` 功能
* `--without-pcre` 禁用 `pcre` 库
* `--with-pcre` 启用 `pcre` 库
* `--with-pcre=` 指向 `pcre` 库文件目录
* `--with-pcre-opt=` 在编译时为 `pcre` 库设置附加参数
* `--with-zlib=` 指向 `zlib` 库文件目录
* `--with-zlib-opt=` 在编译时为 `zlib` 设置附加参数
* `--with-zlib-asm=` 为指定的 `CPU` 使用汇编源进行优化
* `--with-openssl=` 指向 `openssl` 安装目录
* `--with-openssl-opt=` 在编译时为 `openssl` 设置附加参数
* `--with-debug` 启用 `debug` 日志
* `--with-http_mp4_module`  启用 `ngx_http_mp4_module` 支持，启用对 `mp4` 类视频文件的支持
* `--with-http_gzip_static_module`  启用 `ngx_http_gzip_static_module` 支持，支持在线实时压缩输出数据流
* `--with-http_random_index_module`  启用 `ngx_http_random_index_module` 支持，从目录中随机挑选一个目录索引
* `--with-http_secure_link_module`  启用 `ngx_http_secure_link_module` 支持，计算和检查要求所需的安全链接网址
* `--with-http_degradation_module`  启用 `ngx_http_degradation_module` 支持允许在内存不足的情况下返回 `204` 或 `444` 代码
* `--with-http_stub_status_module`  启用 `ngx_http_stub_status_module` 支持查看 `nginx` 的状态页

### 可能出现的问题

#### error: fall through

问题现象：

``` linux
/home/ubuntu/nginx/rtmp-module//ngx_rtmp_eval.c: In function ‘ngx_rtmp_eval’:
/home/ubuntu/nginx/rtmp-module//ngx_rtmp_eval.c:160:17: error: this statement may fall through [-Werror=implicit-fallthrough=]
  160 |                 switch (c) {
      |                 ^~~~~~
/home/ubuntu/nginx/rtmp-module//ngx_rtmp_eval.c:170:13: note: here
  170 |             case ESCAPE:
      |             ^~~~
cc1: all warnings being treated as errors
make[1]: *** [objs/Makefile:1349: objs/addon/rtmp-module/ngx_rtmp_eval.o] Error 1
make[1]: Leaving directory '/home/ubuntu/nginx/nginx-1.18.0'
make: *** [Makefile:8: build] Error 2
```

问题原因：

如果 `gcc` 版本号较高，那么源代码的 `switch-case` 块中如果忘了加上 `break`，有可能会报错 `Implicit fallthrough error`。解决方法有几种，比如：

1. 更换 `gcc` 版本，具体是升级还是降级还需测试（实测 `gcc version 4.8.5` 可以正常编译，`gcc version 9.3.0` 编译不通过）。
2. 在报错的 `switch-case` 块中加上 `break`。此时要对代码逻辑理解清楚，否则加上 `break` 有可能会破坏原来的逻辑。
3. 编译时忽略 `Implicit fallthrough error` 这个错误。

笔者采用了第三种方法，即在编译的时候就忽略这个错误。方法如下：

修改 objs/Makefile 的内容：

``` linux
# 找到改行
CFLAGS =  -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -Werror -g
# 修改为如下内容
CFLAGS =  -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -Wno-implicit-fallthrough -Werror -g
# *以上修改需在 configure 之后 make 之前执行
```

重新 `make` 即可。

## 安装完成后的工作

### 加入系统服务和开机启动

#### Systemctl 工具（推荐）

1. 创建 nginx 服务脚本

    ``` bash
    $ sudo vim /lib/systemd/system/nginx.service
    ```

2. 填入以下内容：

    ``` linux
    [Unit]
    Description=The NGINX HTTP and reverse proxy server
    After=syslog.target network-online.target remote-fs.target nss-lookup.target
    Wants=network-online.target

    [Service]
    Type=forking
    PIDFile=/run/nginx.pid
    ExecStartPre=/usr/local/nginx/sbin/nginx -t
    ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
    ExecReload=/usr/local/nginx/sbin/nginx -s reload
    ExecStop=/bin/kill -s QUIT $MAINPID
    PrivateTmp=true

    [Install]
    WantedBy=multi-user.target
    ```

    重新加载 systemctl 服务：

    ``` bash
    $ sudo systemctl daemon-reload
    ```

    至此，我们可以通过系统 sytemctl 命令来控制 Ngnix 的启动与停止：

    ``` bash
    $ sudo systemctl start nginx # 启动 Nginx
    $ sudo systemctl stop nginx # 关闭 Nginx
    $ sudo systemctl restart nginx # 重启 Nginx
    ```

3. 添加开机启动

    ``` bash
    $ sudo systemctl enable nginx
    ```

#### Service 工具

1. 创建 nginx 启动命令脚本

    ``` bash
    $ sudo vim /etc/init.d/nginx
    ```

2. 插入以下内容, 注意修改 `nginx` 和 `NGINX_CONF_FILE` 字段, 匹配自己的安装路径和配置文件路径

    ``` bash
    #!/bin/bash
    # nginx - this script starts and stops the nginx daemon
    #
    # chkconfig: - 85 15
    # description: Nginx is an HTTP(S) server, HTTP(S) reverse
    # proxy and IMAP/POP3 proxy server
    # processname: nginx
    # config: /etc/nginx/nginx.conf
    # config: /etc/sysconfig/nginx
    # pidfile: /var/run/nginx.pid
    # Source function library.
    . /etc/rc.d/init.d/functions
    # Source networking configuration.
    . /etc/sysconfig/network
    # Check that networking is up.
    [ "$NETWORKING" = "no" ] && exit 0

    nginx="/usr/local/nginx/sbin/nginx"
    prog=$(basename $nginx)
    NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"

    [ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx

    lockfile=/var/lock/subsys/nginx

    start() {
        [ -x $nginx ] || exit 5
        [ -f $NGINX_CONF_FILE ] || exit 6
        echo -n $"Starting $prog:"
        daemon $nginx -c $NGINX_CONF_FILE
        retval=$?
        echo
        [ $retval -eq 0 ] && touch $lockfile
        return $retval
    }

    stop() {
        echo -n $"Stopping $prog:"
        killproc $prog -QUIT
        retval=$?
        echo
        [ $retval -eq 0 ] && rm -f $lockfile
        return $retval
        killall -9 nginx
    }

    restart() {
        configtest || return $?
        stop
        sleep 1
        start
    }

    reload() {
        configtest || return $?
        echo -n $"Reloading $prog:"
        killproc $nginx -HUP
        RETVAL=$?
        echo
    }

    force_reload() {
        restart
    }

    configtest() {
        $nginx -t -c $NGINX_CONF_FILE
    }

    rh_status() {
        status $prog
    }

    rh_status_q() {
        rh_status >/dev/null 2>&1
    }

    case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
        ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
    esac
    ```

3. 设置执行权限

    ``` bash
    $ sudo chmod 755 /etc/init.d/nginx
    ```

4. 修改 `NGINX_CONF_FILE` 指向的配置文件 `/usr/local/nginx/conf/nginx.conf`

    ``` bash
    # 找到如下位置：
    #pid        /logs/nginx.pid;
    # 修改为：
    pid        /var/run/nginx.pid;
    ```

    至此，我们可以通过系统 service 命令来控制 Ngnix 的启动与停止：

    ``` bash
    $ sudo service nginx start # 启动 Nginx
    $ sudo service nginx stop # 关闭 Nginx
    $ sudo service nginx restart # 重启 Nginx
    ```

5. 加入自启动

``` bash
$ chkconfig --list # 显示开机自启的服务
$ chkconfig --add /etc/init.d/nginx # 将 Nginx 加入 chkconfig 管理列表
$ chkconfig nginx on # 设置 Nginx 开机自启动
```

### 加入环境变量

在 `/etc/profile` 文件最后加入：

``` bash
export NGINX_HOME=/usr/local/nginx
export PATH=$PATH:$NGINX_HOME/sbin
```

保存，执行 `source /etc/profile`，使配置文件生效。

然后执行 `nginx -v` 能够正常返回，即为设置成功。

## 参考资料

* [1] 机械工业出版社 《深入理解 Nginx：模块开发与架构解析 第二版》
* [2] 中国工信出版集团 《直播系统开发 基于 Nginx 与 Nginx-rtmp-module》
* [3] [Nginx 配置开机启动 /etc/init.d/nginx - 简书](https://www.jianshu.com/p/0b52f8f3e787)
