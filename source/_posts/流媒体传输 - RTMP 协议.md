---
title: 流媒体传输 - RTMP 协议
categories:
    - 流媒体传输
tags:
  - RTMP
abbrlink: ccfdbe65
date: 2020-10-12 13:39:55
---
RTMP 是 Real Time Messaging Protocol（实时消息传输协议）的首字母缩写。它是由 Adobe 公司提出的一种应用层的协议，用来解决多媒体数据传输流的多路复用（Multiplexing）和分包（packetizing）的问题。

该协议基于 TCP，是一个协议族，包括 RTMP 基本协议及 RTMPT/RTMPS/RTMPE 等多种变种。

RTMP 是一种设计用来进行实时数据通信的网络协议，主要用来在 Flash/AIR 平台和支持 RTMP 协议的流媒体或交互服务器之间进行音视频和数据通信。

<!-- more -->

## 协议介绍

RTMP 协议规定，播放一个流媒体有两个前提步骤：第一步，建立一个网络连接（NetConnection）；第二步，建立一个网络流（NetStream）。

其中，网络连接代表服务器端应用程序和客户端之间基础的连通关系。网络流代表了发送多媒体数据的通道。服务器和客户端之间只能建立一个网络连接，但是基于该连接可以创建很多网络流。他们的关系如图所示：

![RTMP 中的逻辑结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201012214637.png)

播放一个 RTMP 协议的流媒体需要经过以下几个步骤：握手，建立连接，建立流，播放，四个步骤流程如下：

                +----------+                     +----------+
                |  Client  |   TCP/IP Network    |  Server  |
                +----------+                     +----------+
    --------+---|                     |                     |
            |Uninitialized            |            Uninitialized
            |   |----------C0-------->|                     |
            |   |                     |---------C0--------->|
            |   |----------C1-------->|                     |
            |   |                     |<--------S0----------|
            |Version sent             |<--------S1----------|
    Hand-   |   |<---------S0---------|                     |
    shake   |   |<---------S1---------|             Version sent
            |   |                     |---------C1--------->|
            |   |----------C2-------->|                     |
            |   |                     |<--------S2----------|
            |Ack Sent                 |                Ack Sent
            |   |<---------S2---------|                     |
            |   |                     |---------C2--------->|
            |Handshake Done           |           Handshake Done
    --------+---|                                           |
                |                                           |
    --------+---|---------- Command Message(connect) ------>|
            |   |                                           |
            |   |<------ Window Acknowledgement Size -------|
            |   |                                           |
            |   |<---------- Set Peer Bandwidth ------------|
            |   |                                           |
    Appli-  |   |------- Window Acknowledgement Size ------>|
    cation  |   |                                           |
    Connect |   |<---- User Control Message(StreamBegin) ---|
            |   |                                           |
            |   |<----------- Command Message --------------|
            |   |       (_result- connect response)         |
    --------+---|                                           |
                |                                           |
    --------+---|------ Command Message(createStream) ----->|
    Create  |   |                                           |
    Stream  |   |<----------- Command Message --------------|
    --------+---|      (_result- createStream response)     |
                |                                           |
    --------+---|--------- Command Message (play) --------->|
            |   |                                           |
            |   |<-------------- SetChunkSize --------------|
            |   |                                           |
            |   |<---- User Control (StreamIsRecorded) -----|
            |   |                                           |
            |   |<------- UserControl (StreamBegin) --------|
            |   |                                           |
    Play    |   |<-- Command Message(onStatus-play reset) --|
            |   |                                           |
            |   |<-- Command Message(onStatus-play start) --|
            |   |                                           |
            |   |<------------- Audio Message --------------|
            |   |                                           |
            |   |<------------- Video Message --------------|
            |   |                     |                     |
                                      |
               Keep receiving audio and video stream till finishes

## 握手（HandShake）

### 握手交互流程

一个 RTMP 连接以握手开始，双方分别发送大小固定的三个数据块：

1. 握手开始于客户端发送 C0、C1 块。服务器收到 C0 或 C1 后发送 S0 和 S1。
2. 当客户端收齐 S0 和 S1 后，开始发送 C2。当服务器收齐 C0 和 C1 后，开始发送 S2。
3. 当客户端和服务器分别收到 S2 和 C2 后，握手完成。

                    +----------+                     +----------+
                    |  Client  |   TCP/IP Network    |  Server  |
                    +----------+                     +----------+
        --------+---|                     |                     |
                |Uninitialized            |            Uninitialized
                |   |----------C0-------->|                     |
                |   |                     |---------C0--------->|
                |   |----------C1-------->|                     |
                |   |                     |<--------S0----------|
                |Version sent             |<--------S1----------|
        Hand-   |   |<---------S0---------|                     |
        shake   |   |<---------S1---------|             Version sent
                |   |                     |---------C1--------->|
                |   |----------C2-------->|                     |
                |   |                     |<--------S2----------|
                |Ack Sent                 |                Ack Sent
                |   |<---------S2---------|                     |
                |   |                     |---------C2--------->|
                |Handshake Done           |           Handshake Done
        --------+---|                                           |
                        Pictorial Representation of Handshake

### 简化的握手交互流程

而在实际应用中，通常使用三次握手来简化以上握手流程：

1. Client ---- C0&C1 ----> Server
2. Client <---- S0&S1&S2 ---- Server
3. Client ---- C2 ----> Server

### 握手交互抓包

通过 Wirshark 抓包，得到以下数据包：

1. Client ---- C0&C1 ----> Server

        0000   03 00 00 00 00 09 00 7c 02 f7 78 55 1e ce ab 8e   .......|..xU....
        0010   1e 36 2f 07 c5 86 8a 70 b2 66 d4 02 20 e5 08 61   .6/....p.f.. ..a
        ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
        05f0   c4 10 96 07 08 84 39 34 53 ce 50 96 94 af be ab   ......94S.P.....
        0600   e0                                                .
        -------------------------------------------------------------------------
        Protocol version: 03
        Handshake C1: 0000000009007c02f778551eceab8e1e362f07c5868a70b2…

2. Client <---- S0&S1&S2 ---- Server

        0000   03 06 a7 7f 3b 0d 0e 0a 0d 8b 4c 51 8d d0 a9 c7   ....;.....LQ....
        0010   21 e8 6a 5b b0 4a 9d 74 91 0f 30 03 fa bc 77 6e   !.j[.J.t..0...wn
        ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
        0600   2d a1 d7 ca 81 18 80 02 9a d4 94 92 ab 14 ea 8a   -...............
        ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
        0bf0   96 4b 7e 0c f1 27 fb fd 52 02 19 5d d7 73 57 47   .K~..'..R..].sWG
        0c00   f0                                                .
        -------------------------------------------------------------------------
        Protocol version: 03
        Handshake S1: 06a77f3b0d0e0a0d8b4c518dd0a9c721e86a5bb04a9d7491…
        Handshake S2: a1d7ca811880029ad49492ab14ea8a119cd415aa26c62ee4…

3. Client ---- C2 ----> Server

        0000   e2 5c 15 c2 2c af 72 d0 98 6f bd 3e da 0d 71 51   .\..,.r..o.>..qQ
        ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
        05f0   94 e0 bc b0 94 31 f4 20 d6 e7 d3 03 6d 2c 71 77   .....1. ....m,qw
        -------------------------------------------------------------------------
        Handshake C2: e25c15c22caf72d0986fbd3eda0d7151973e19083e0abbef…

### 握手报文分析

#### C0 与 S0

        C0 与 S0
    +-+-+-+-+-+-+-+-+
    |    version    |
    +-+-+-+-+-+-+-+-+

- C0：客户端发送其所支持的 RTMP 版本号：3~31。一般都是写 3
- S1：服务端返回其所支持的版本号。如果没有客户端的版本号，默认返回 3

#### C1 与 S1

           C1 与 S1
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   zero (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+

- C1/S1 长度为 1536B。主要目的是确保握手的唯一性。
- 格式为 time + zero + random
- time 发送时间戳，长度 4 byte
- zero 保留值 0，长度 4 byte
- random 随机值，长度 1528 byte，保证此次握手的唯一性，确定握手的对象

#### C2 与 S2

           C2 与 S2
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   time2(4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+

- C2/S2 的长度也是 1536B。相当于就是 S1/C1 的响应值，对应 C1/S1 的 Copy 值，在于字段有点区别
- time， C2/S2 发送的时间戳，长度 4 byte
- time2， S1/C1 发送的时间戳，长度 4 byte
- random，S1/C1 发送的随机数，长度为 1528B

## 建立网络连接（NetConnection）

### 建立网络连接交互流程

1. Client ----> Server : Command Message Connect
2. Client <---- Server : Window Acknowledgement Size
3. Client <---- Server : Set Peer Bandwidth
4. Client ----> Server : Window Acknowledgement Size
5. Client <---- Server : User Control Message Stream Begin
6. Client <---- Server : Command Message _result

                    +----------+                     +----------+
                    |  Client  |          |          |  Server  |
                    +----------+          |          +----------+
        --------+---|---------- Command Message(connect) ------>|
                |   |                                           |
                |   |<------ Window Acknowledgement Size -------|
                |   |                                           |
                |   |<---------- Set Peer Bandwidth ------------|
                |   |                                           |
        Appli-  |   |------- Window Acknowledgement Size ------>|
        cation  |   |                                           |
        Connect |   |<---- User Control Message(StreamBegin) ---|
                |   |                                           |
                |   |<----------- Command Message --------------|
                |   |       (_result- connect response)         |
        --------+---|                                           |
                        Message flow in the connect command

### 建立网络连接抓包

1. Client ----> Server : AMF0 Command connect()

        0000   03 00 00 00 00 00 cc 14 00 00 00 00 02 00 07 63   ...............c
        0010   6f 6e 6e 65 63 74 00 3f f0 00 00 00 00 00 00 03   onnect.?........
        0020   00 03 61 70 70 02 00 06 6d 79 6c 69 76 65 00 08   ..app...mylive..
        0030   66 6c 61 73 68 56 65 72 02 00 0d 4c 4e 58 20 39   flashVer...LNX 9
        0040   2c 30 2c 31 32 34 2c 32 00 05 74 63 55 72 6c 02   ,0,124,2..tcUrl.
        0050   00 20 72 74 6d 70 3a 2f 2f 36 32 2e 32 33 34 2e   . rtmp://62.234.
        0060   31 31 31 2e 31 33 3a 31 39 33 35 2f 6d 79 6c 69   111.13:1935/myli
        0070   76 65 00 04 66 70 61 64 01 00 00 0c 63 61 70 61   ve..fpad....capa
        0080   62 69 6c 69 74 69 65 73 00 40 2e 00 00 00 00 00   bilities.@......
        0090   00 00 0b 61 75 64 69 6f 43 6f 64 65 63 73 00 40   ...audioCodecs.@
        00a0   af ce 00 00 00 00 00 00 0b 76 69 64 65 6f 43 6f   .........videoCo
        00b0   64 65 63 73 00 40 6f 80 00 00 00 00 00 00 0d 76   decs.@o........v
        00c0   69 64 65 6f 46 75 6e 63 74 69 6f 6e 00 3f f0 00   ideoFunction.?..
        00d0   00 00 00 00 00 00 00 09                           ........
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command connect('mylive'))
            Response to this call in frame: 20
            RTMP Header
                00.. .... = Format: 0
                ..00 0011 = Chunk Stream ID: 3
                Timestamp: 0
                Body size: 204
                Type ID: AMF0 Command (0x14)
                Stream ID: 0
            RTMP Body
                String 'connect'
                Number 1
                Object (8 items)
                    AMF0 type: Object (0x03)
                    Property 'app' String 'mylive'
                    Property 'flashVer' String 'LNX 9,0,124,2'
                    Property 'tcUrl' String 'rtmp://62.234.111.13:1935/mylive'
                    Property 'fpad' Boolean false
                    Property 'capabilities' Number 15
                    Property 'audioCodecs' Number 4071
                    Property 'videoCodecs' Number 252
                    Property 'videoFunction' Number 1
                    End Of Object Marker

2. Server ----> Client : Window Acknowledgement Size

        0000   02 00 00 00 00 00 04 05 00 00 00 00 00 4c 4b 40   .............LK@
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (Window Acknowledgement Size 5000000)
            RTMP Header
                00.. .... = Format: 0
                ..00 0010 = Chunk Stream ID: 2
                Timestamp: 0
                Body size: 4
                Type ID: Window Acknowledgement Size (0x05)
                Stream ID: 0
            RTMP Body
                Window acknowledgement size: 5000000

3. Server ----> Client : Set Peer Bandwidth

        0000   02 00 00 00 00 00 05 06 00 00 00 00 00 4c 4b 40   .............LK@
        0010   02                                                .
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (Set Peer Bandwidth 5000000,Dynamic)
            RTMP Header
                00.. .... = Format: 0
                ..00 0010 = Chunk Stream ID: 2
                Timestamp: 0
                Body size: 5
                Type ID: Set Peer Bandwidth (0x06)
                Stream ID: 0
            RTMP Body
                Window acknowledgement size: 5000000
                Limit type: Dynamic (2)

4. Server ----> Client : Set Chunk Size

        0000   02 00 00 00 00 00 04 01 00 00 00 00 00 00 10 00   ................
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (Set Chunk Size 4096)
            RTMP Header
                00.. .... = Format: 0
                ..00 0010 = Chunk Stream ID: 2
                Timestamp: 0
                Body size: 4
                Type ID: Set Chunk Size (0x01)
                Stream ID: 0
            RTMP Body
                Chunk size: 4096

5. Server ----> Client : AMF0 Command _result

        0000   03 00 00 00 00 00 be 14 00 00 00 00 02 00 07 5f   ..............._
        0010   72 65 73 75 6c 74 00 3f f0 00 00 00 00 00 00 03   result.?........
        0020   00 06 66 6d 73 56 65 72 02 00 0d 46 4d 53 2f 33   ..fmsVer...FMS/3
        0030   2c 30 2c 31 2c 31 32 33 00 0c 63 61 70 61 62 69   ,0,1,123..capabi
        0040   6c 69 74 69 65 73 00 40 3f 00 00 00 00 00 00 00   lities.@?.......
        0050   00 09 03 00 05 6c 65 76 65 6c 02 00 06 73 74 61   .....level...sta
        0060   74 75 73 00 04 63 6f 64 65 02 00 1d 4e 65 74 43   tus..code...NetC
        0070   6f 6e 6e 65 63 74 69 6f 6e 2e 43 6f 6e 6e 65 63   onnection.Connec
        0080   74 2e 53 75 63 63 65 73 73 00 0b 64 65 73 63 72   t.Success..descr
        0090   69 70 74 69 6f 6e 02 00 15 43 6f 6e 6e 65 63 74   iption...Connect
        00a0   69 6f 6e 20 73 75 63 63 65 65 64 65 64 2e 00 0e   ion succeeded...
        00b0   6f 62 6a 65 63 74 45 6e 63 6f 64 69 6e 67 00 00   objectEncoding..
        00c0   00 00 00 00 00 00 00 00 00 09                     ..........
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command _result('NetConnection.Connect.Success'))
            Call for this response in frame: 17
            RTMP Header
                00.. .... = Format: 0
                ..00 0011 = Chunk Stream ID: 3
                Timestamp: 0
                Body size: 190
                Type ID: AMF0 Command (0x14)
                Stream ID: 0
            RTMP Body
                String '_result'
                Number 1
                Object (2 items)
                    AMF0 type: Object (0x03)
                    Property 'fmsVer' String 'FMS/3,0,1,123'
                    Property 'capabilities' Number 31
                    End Of Object Marker
                Object (4 items)
                    AMF0 type: Object (0x03)
                    Property 'level' String 'status'
                    Property 'code' String 'NetConnection.Connect.Success'
                    Property 'description' String 'Connection succeeded.'
                    Property 'objectEncoding' Number 0
                    End Of Object Marker

6. Client ----> Server : Window Acknowledgement Size

        0000   02 00 00 00 00 00 04 05 00 00 00 00 00 4c 4b 40   .............LK@
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (Window Acknowledgement Size 5000000)
            RTMP Header
                00.. .... = Format: 0
                ..00 0010 = Chunk Stream ID: 2
                Timestamp: 0
                Body size: 4
                Type ID: Window Acknowledgement Size (0x05)
                Stream ID: 0
            RTMP Body
                Window acknowledgement size: 5000000

## 建立网络流（NetStream）

1. 客户端发送命令消息中的“创建流”（createStream）命令到服务器端。
2. 服务器端接收到“创建流”命令后，发送命令消息中的“结果”(_result)，通知客户端流的状态。

                    +----------+                     +----------+
                    |  Client  |          |          |  Server  |
                    +----------+          |          +----------+
        --------+---|------ Command Message(createStream) ----->|
        Create  |   |                                           |
        Stream  |   |<----------- Command Message --------------|
        --------+---|      (_result- createStream response)     |

### 建立网络流抓包

1. Client ----> Server : AMF0 Command createStream()

        0000   43 00 00 00 00 00 19 14 02 00 0c 63 72 65 61 74   C..........creat
        0010   65 53 74 72 65 61 6d 00 40 00 00 00 00 00 00 00   eStream.@.......
        0020   05                                                .
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command createStream())
            Response to this call in frame: 25
            RTMP Header
                01.. .... = Format: 1
                ..00 0011 = Chunk Stream ID: 3
                Timestamp delta: 0
                Timestamp: 0 (calculated)
                Body size: 25
                Type ID: AMF0 Command (0x14)
            RTMP Body
                String 'createStream'
                Number 2
                Null

2. Server ----> Client : AMF0 Command _result

        0000   03 00 00 00 00 00 1d 14 00 00 00 00 02 00 07 5f   ..............._
        0010   72 65 73 75 6c 74 00 40 00 00 00 00 00 00 00 05   result.@........
        0020   00 3f f0 00 00 00 00 00 00                        .?.......
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command _result())
            Call for this response in frame: 23
            RTMP Header
                00.. .... = Format: 0
                ..00 0011 = Chunk Stream ID: 3
                Timestamp: 0
                Body size: 29
                Type ID: AMF0 Command (0x14)
                Stream ID: 0
            RTMP Body
                String '_result'
                Number 2
                Null
                Number 1

## 播放（Play）

1. 客户端发送命令消息中的播放（play）命令到服务器。
2. 接收到播放命令后，服务器发送设置块大小（ChunkSize）协议消息。
3. 服务器发送用户控制消息中的 streambegin，告知客户端流ID。
4. 播放命令成功的话，服务器发送命令消息中的“响应状态” NetStream.Play.Start & NetStream.Play.reset，告知客户端播放命令执行成功。
5. 在此之后服务器发送客户端要播放的音频和视频数据。

                +----------+                     +----------+
                |  Client  |          |          |  Server  |
                +----------+          |          +----------+
    --------+---|--------- Command Message (play) --------->|
            |   |                                           |
            |   |<-------------- SetChunkSize --------------|
            |   |                                           |
            |   |<---- User Control (StreamIsRecorded) -----|
            |   |                                           |
            |   |<------- UserControl (StreamBegin) --------|
            |   |                                           |
    Play    |   |<-- Command Message(onStatus-play reset) --|
            |   |                                           |
            |   |<-- Command Message(onStatus-play start) --|
            |   |                                           |
            |   |<------------- Audio Message --------------|
            |   |                                           |
            |   |<------------- Video Message --------------|
            |   |                     |                     |
                                      |
               Keep receiving audio and video stream till finishes

### 播放抓包

1. Client ----> Server : Command getStreamLength()

        0000   08 00 00 00 00 00 1f 14 00 00 00 00 02 00 0f 67   ...............g
        0010   65 74 53 74 72 65 61 6d 4c 65 6e 67 74 68 00 40   etStreamLength.@
        0020   08 00 00 00 00 00 00 05 02 00 00                  ...........
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command getStreamLength())
            RTMP Header
                00.. .... = Format: 0
                ..00 1000 = Chunk Stream ID: 8
                Timestamp: 0
                Body size: 31
                Type ID: AMF0 Command (0x14)
                Stream ID: 0
            RTMP Body
                String 'getStreamLength'
                Number 3
                Null
                String ''

2. Client ----> Server : Command play('')

        0000   08 00 00 00 00 00 1d 14 01 00 00 00 02 00 04 70   ...............p
        0010   6c 61 79 00 40 10 00 00 00 00 00 00 05 02 00 00   lay.@...........
        0020   00 c0 9f 40 00 00 00 00 00                        ...@.....
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command play(''))
            RTMP Header
                00.. .... = Format: 0
                ..00 1000 = Chunk Stream ID: 8
                Timestamp: 0
                Body size: 29
                Type ID: AMF0 Command (0x14)
                Stream ID: 1
            RTMP Body
                String 'play'
                Number 4
                Null
                String ''
                Number -2000

3. Client ----> Server : User Control Message Set Buffer Length

        0000   42 00 00 01 00 00 0a 04 00 03 00 00 00 01 00 00   B...............
        0010   0b b8                                             ..
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (User Control Message Set Buffer Length 1,3000ms)
            RTMP Header
                01.. .... = Format: 1
                ..00 0010 = Chunk Stream ID: 2
                Timestamp delta: 1
                Timestamp: 1 (calculated)
                Body size: 10
                Type ID: User Control Message (0x04)
            RTMP Body
                Event type: Set Buffer Length (3)

4. Server ----> Client : User Control Message Stream Begin

        0000   02 00 00 00 00 00 06 04 00 00 00 00 00 00 00 00   ................
        0010   00 01                                             ..
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (User Control Message Stream Begin 1)
            RTMP Header
                00.. .... = Format: 0
                ..00 0010 = Chunk Stream ID: 2
                Timestamp: 0
                Body size: 6
                Type ID: User Control Message (0x04)
                Stream ID: 0
            RTMP Body
                Event type: Stream Begin (0)

5. Server ----> Client : Command onStatus('NetStream.Play.Start')

        0000   05 00 00 00 00 00 60 14 01 00 00 00 02 00 08 6f   ......`........o
        0010   6e 53 74 61 74 75 73 00 00 00 00 00 00 00 00 00   nStatus.........
        0020   05 03 00 05 6c 65 76 65 6c 02 00 06 73 74 61 74   ....level...stat
        0030   75 73 00 04 63 6f 64 65 02 00 14 4e 65 74 53 74   us..code...NetSt
        0040   72 65 61 6d 2e 50 6c 61 79 2e 53 74 61 72 74 00   ream.Play.Start.
        0050   0b 64 65 73 63 72 69 70 74 69 6f 6e 02 00 0a 53   .description...S
        0060   74 61 72 74 20 6c 69 76 65 00 00 09               tart live...
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Command onStatus('NetStream.Play.Start'))
            RTMP Header
                00.. .... = Format: 0
                ..00 0101 = Chunk Stream ID: 5
                Timestamp: 0
                Body size: 96
                Type ID: AMF0 Command (0x14)
                Stream ID: 1
            RTMP Body
                String 'onStatus'
                Number 0
                Null
                Object (3 items)

6. Server ----> Client : Data |RtmpSampleAccess()

        0000   05 00 00 00 00 00 18 12 01 00 00 00 02 00 11 7c   ...............|
        0010   52 74 6d 70 53 61 6d 70 6c 65 41 63 63 65 73 73   RtmpSampleAccess
        0020   01 01 01 01                                       ....
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Data |RtmpSampleAccess())
            RTMP Header
                00.. .... = Format: 0
                ..00 0101 = Chunk Stream ID: 5
                Timestamp: 0
                Body size: 24
                Type ID: AMF0 Data (0x12)
                Stream ID: 1
            RTMP Body
                String '|RtmpSampleAccess'
                Boolean true
                Boolean true

7. Server ----> Client : Data onMetaData()

        0000   05 00 00 00 00 01 83 12 01 00 00 00 02 00 0a 6f   ...............o
        0010   6e 4d 65 74 61 44 61 74 61 03 00 06 53 65 72 76   nMetaData...Serv
        ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
        0180   00 00 00 00 00 00 00 00 00 00 00 00 00 00 09      ...............
        -------------------------------------------------------------------------
        Real Time Messaging Protocol (AMF0 Data onMetaData())
            RTMP Header
                00.. .... = Format: 0
                ..00 0101 = Chunk Stream ID: 5
                Timestamp: 0
                Body size: 387
                Type ID: AMF0 Data (0x12)
                Stream ID: 1
            RTMP Body
                String 'onMetaData'
                Object (14 items)
                    AMF0 type: Object (0x03)
                    Property 'Server' String 'NGINX RTMP (github.com/arut/nginx-rtmp-module)'
                    Property 'width' Number 1280
                    Property 'height' Number 720
                    Property 'displayWidth' Number 1280
                    Property 'displayHeight' Number 720
                    Property 'duration' Number 0
                    Property 'framerate' Number 25
                    Property 'fps' Number 25
                    Property 'videodatarate' Number 0
                    Property 'videocodecid' Number 7
                    Property 'audiodatarate' Number 125
                    Property 'audiocodecid' Number 1
                    Property 'profile' String ''
                    Property 'level' String ''
                    End Of Object Marker

## 数据传输

### 音频数据

    0000   80 fa 5b 46 e3 29 dc a6 32 90 25 1e 08 00 45 00   ..[F.)..2.%...E.
    ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
    05e0   aa aa 02 ab 2a 26 a8 20 06 60                     ....*&. .`
    -------------------------------------------------------------------------
    Real Time Messaging Protocol (Audio Data)
        RTMP Header
            00.. .... = Format: 0
            ..00 0110 = Chunk Stream ID: 6
            Timestamp: 59947
            Body size: 2054
            Type ID: Audio Data (0x08)
            Stream ID: 1
        RTMP Body
            Control: 0x1f (ADPCM 44 kHz 16 bit stereo)
                0001 .... = Format: ADPCM (1)
                .... 11.. = Sample rate: 44 kHz (3)
                .... ..1. = Sample size: 16 bit (1)
                .... ...1 = Channels: stereo (1)
            Audio data: 80a1a80292a0884488cccd0c9088cd54cc88888888c8ccd0…

### 视频数据

    0000   47 00 00 28 00 1a 1b 09 27 01 00 00 a0 00 00 1a   G..(....'.......
    ····   ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ·· ··   ················
    1a20   b4 9e 58                                          ..X
    -------------------------------------------------------------------------
    Real Time Messaging Protocol (Video Data)
        RTMP Header
            01.. .... = Format: 1
            ..00 0111 = Chunk Stream ID: 7
            Timestamp delta: 40
            Timestamp: 129960 (calculated)
            Body size: 6683
            Type ID: Video Data (0x09)
        RTMP Body
            Control: 0x27 (inter-frame H.264)
                0010 .... = Type: inter-frame (2)
                .... 0111 = Format: H.264 (7)
            Video data: 010000a000001a12419aab49a8416c994c085ffffe8d340d…

## 参考资料

* [1] [《Adobe’s Real Time Messaging Protocol》](https://wwwimages2.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
* [2] [RTMP 协议详解 - 季末的天堂 - 博客园](https://www.cnblogs.com/jimodetiantang/p/8974075.html)
