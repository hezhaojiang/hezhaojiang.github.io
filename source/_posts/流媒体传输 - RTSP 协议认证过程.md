---
title: 流媒体传输 - RTSP 协议认证过程
categories:
  - 流媒体传输
tags:
  - RTSP
abbrlink: dc673a79
date: 2020-10-15 15:24:33
---
Rtsp 认证 主要分为两种：

基本认证 （Basic authentication）和 摘要认证 （Digest authentication）

基本认证是 HTTP 1.0 提出的认证方案，其消息传输不经过加密转换因此存在严重的安全隐患。

摘要认证是 HTTP 1.1 提出的基本认证的替代方案，其消息经过 MD5 哈希转换因此具有更高的安全性。

<!--more-->

## 基本认证

1. 客户端发送 DESCRIBE 请求到服务端

    ``` node
    DESCRIBE rtsp://192.168.199.242:554/ch1/main/av_stream RTSP/1.0
    CSeq: 3
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Accept: application/sdp
    ```

2. RTSP 服务端认为没有通过认证，发出 WWW-Authenticate 认证响应

        WWW-Authenticate 中应携带有 Basic 字样、realm 字段

    ``` node
    RTSP/1.0 401 Unauthorized
    CSeq: 3
    WWW-Authenticate: Basic realm="IP Camera(D1846)"
    Date:  Thu, Oct 15 2020 23:30:55 GMT
    ```

3. 客户端携带 Authorization 串再次发出 DESCRIBE 请求

        Authorization 串计算方法：
        Authorization = base64(username:password)
        
        * 用户名：admin * 密码：Abc12345
        Authorization = base64(admin:Abc12345)
                      = YWRtaW46QWJjMTIzNDU=

    ``` node
    DESCRIBE rtsp://192.168.199.242:554/ch1/main/av_stream RTSP/1.0
    CSeq: 4
    Authorization: Basic YWRtaW46QWJjMTIzNDU=
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Accept: application/sdp
    ```

4. 服务器对客户端反馈的 Authorization 进行校验，通过则返回 200 OK

    ``` node
    RTSP/1.0 200 OK
    CSeq: 4
    Content-Type: application/sdp
    Content-Base: rtsp://192.168.199.242:554/ch1/main/av_stream/
    Content-Length: 594
    ```

## 摘要认证

1. 客户端发送 DESCRIBE 请求到服务端

    ``` node
    DESCRIBE rtsp://192.168.199.242:554/ch1/main/av_stream RTSP/1.0
    CSeq: 3
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Accept: application/sdp
    ```

2. RTSP 服务端认为没有通过认证，发出 WWW-Authenticate 认证响应

        WWW-Authenticate 中应携带有 Digest 字样、realm 字段、nonce 字段

    ``` node
    RTSP/1.0 401 Unauthorized
    CSeq: 3
    WWW-Authenticate: Digest realm="IP Camera(D1846)", nonce="61f92652b25e740d73887108b419e8b6", stale="FALSE"
    Date:  Thu, Oct 15 2020 23:30:55 GMT
    ```

3. 客户端以 用户名、密码、nonce、RTSP 方法、请求的 URI 等信息为基础产生 response 信息进行反馈

        response 计算方法：
        RTSP 客户端应该使用 username + password 并计算 response 如下:
        如果 password 为 MD5 编码, 则
            response = md5(password:nonce:md5(public_method:url));
        如果 password 为 ANSI 字符串, 则
            response = md5(md5(username:realm:password):nonce:md5(public_method:url));

        * 用户名：admin * 密码：Abc12345
        response = md5(md5(admin:IP Camera(D1846):Abc12345):61f92652b25e740d73887108b419e8b6:md5(DESCRIBE:rtsp://192.168.199.242:554/ch1/main/av_stream));
                 = md5(e03ca5323610c45d55574699008d1c34:61f92652b25e740d73887108b419e8b6:034c626594db40bb96121247d4492461)
                 = 58ad47d8436ca5db7356e6a084abcbf9

    ``` node
    DESCRIBE rtsp://192.168.199.242:554/ch1/main/av_stream RTSP/1.0
    CSeq: 4
    Authorization: Digest username="admin", realm="IP Camera(D1846)", nonce="61f92652b25e740d73887108b419e8b6", uri="rtsp://192.168.199.242:554/ch1/main/av_stream", response="58ad47d8436ca5db7356e6a084abcbf9"
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Accept: application/sdp
    ```

4. 服务器对客户端反馈的 response 进行校验，通过则返回 200 OK

    ``` node
    RTSP/1.0 200 OK
    CSeq: 4
    Content-Type: application/sdp
    Content-Base: rtsp://192.168.199.242:554/ch1/main/av_stream/
    Content-Length: 594
    ```
