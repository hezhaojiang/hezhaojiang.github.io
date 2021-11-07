---
title: 流媒体传输 - RTSP Over HTTPS
categories:
  - 流媒体传输
tags:
  - RTSP
abbrlink: '880163e6'
date: 2020-09-15 13:39:43
---
{% post_link "流媒体传输 - RTSP Over HTTP" %} 中我们分析了如何使用 HTTP 来进行 RTSP 交互，而在一些使用场景中，RTSP 和 HTTP 的明文交互带来了很多安全性问题，而 HTTPS 的出现，大大提高了数据交互的安全性，而在 RTSP Over HTTP 协议中，我们也可以参考 HTTP 和 HTTPS 的修改，实现 RTSP Over HTTPS 协议，从而可以应用在一些安全性要求较高的场景中。

<!-- more -->

## 基础知识

### HTTP 和 HTTPS

详见：

- {% post_link "HTTP 和 HTTPS 协议" %}
- [The First Few Milliseconds of an HTTPS Connection](http://www.moserware.com/2009/06/first-few-milliseconds-of-https.html)

## 参考资料
