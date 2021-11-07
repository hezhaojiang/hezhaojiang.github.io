---
title: 善用佳软 - Astyle 安装与使用
categories:
  - 工具软件
tags:
  - Astyle
abbrlink: e2bb409
date: 2020-07-23 15:12:40
---

```bash
$ Astyle --style=ansi --indent=spaces=4 -L -xC100 -V -f -z2 -H -p -U --suffix=none --quiet
# --style=ansi
# --indent=spaces=4 更改缩进 4 个空格
# -L 缩进label，让 label 比当前的内容先前一个缩进距离，而不是通通靠左
# -xC100 最长80个字符
# -V 将tab转换为空格
# -f 在两行不相关的代码之间插入空行
# -H 在if for等关键字后面，加一个空格
# -p 在操作符两边加空格
# -U 去掉()内部不必要的空格
# --suffix=none 不备份原始文件
```
