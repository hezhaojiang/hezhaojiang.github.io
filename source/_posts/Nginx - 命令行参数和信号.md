---
title: Nginx - 命令行参数和信号
tags:
  - Nginx
abbrlink: f45df2c9
date: 2020-10-07 05:45:16
---
## nginx 命令行参数

``` linux
[root@VM_0_6_centos ~]# nginx -?
nginx version: nginx/1.18.0
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/local/nginx/)
  -c filename   : set configuration file (default: conf/nginx.conf)
  -g directives : set global directives out of configuration file
```

<!--more-->

### nginx [-?hvVtTq]

``` bash
# 打开帮助
nginx -h/-?
# 打印 nginx 的版本信息
nginx -v
# 打印 nginx 的版本和编译信息
nginx -V
# 测试配置文件是否有语法错误
nginx -t
# 测试配置文件，转储并退出
nginx -T
# 检测配置文件时屏蔽非错误信息，制输出错误信息
nginx -q
```

### nginx [-s signal]

``` bash
# 重新开始记录日志文件
nginx -s reopen
# 立刻停止服务
nginx -s stop
# 优雅的停止服务
nginx -s quit
# 重载加载配置文件并重启（平滑重启）
nginx -s reload
```

### nginx [-c filename]

``` bash
# 使用指定的配置文件，而不是默认的 conf 文件夹下的配置文件
nginx -c /usr/local/nginx/conf/nginx.conf
```

### nginx [-p prefix]

``` bash
# 指定运行目录
nginx -p /usr/local/nginx
```

### nginx [-g directives]

``` bash
# 指定配置命令，覆盖掉配置文件中的指令
nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw,ncpu`;"
```

## nginx 信号控制

### 常见信号控制

`nginx` 主进程可以处理以下的信号：

| 信号 | 说明 |
| - | - |
| `TERM`, `INT` | 快速关闭 |
| `QUIT` | 从容关闭，等待请求结束后关闭 |
| `HUP` | 重载配置，用新的配置开始新的工作进程，从容关闭旧的工作进程 |
| `USR1` | 重新打开日志文件，用于切割日志 |
| `USR2` | 平滑升级可执行程序 |
| `WINCH` | 从容关闭工作进程，可配合 USR2 信号平滑升级可执行程序 |

### 使用信号控制

信号控制的具体语法为： `kill` `-信号选项` `Nginx 主进程号`，例如：

```bash
kill -INT 主进程号 # 关闭 Nginx 进程
kill -HUP 主进程号 # 平滑地读取新配置文件，不必重启 Nginx
kill -USR1 主进程号
kill -USR2 主进程号
kill -WINCH 主进程号
```

## 参考资料

* [1] 中国工信出版集团 《直播系统开发 基于 Nginx 与 Nginx-rtmp-module》
