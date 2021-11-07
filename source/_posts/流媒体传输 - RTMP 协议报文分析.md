---
title: 流媒体传输 - RTMP 协议报文分析
categories:
    - 流媒体传输
tags:
  - RTMP
abbrlink: 50f2e42d
date: 2020-10-12 15:26:42
---
握手之后，连接开始对一个或多个 `chunk stream` 进行合并。创建的每个块都有一个唯一 `id` 对其进行关联，这个 `id` 叫做 `chunk stream id`。这些块通过网络进行传输。传递时，每个块必须被完全发送才可以发送下一块。在接收端，这些块被根据 `chunk stream id` 被组装成消息。

每个块包含一个头和数据体。块头包含三个部分：

    +--------------+----------------+--------------------+--------------+
    | Basic Header | Message Header | Extended Timestamp |  Chunk Data  |
    +--------------+----------------+--------------------+--------------+
    |<------------------- Chunk Header ----------------->|
                                Chunk Format

<!-- more -->

## Chunk Header

`chunk Header` 包括 `Basic Header`、`Message Header`、`Extended Timestamp` 三部分。

- `Basic Header` (1 - 3 bits)：这个字段对 `chunk stream id` 和块类型进行编码。块类型决定了消息头的编码格式。该字段长度完全取决于 `chunk stream id`，因为 `chunk stream id` 是一个可变长度的字段。
- `Message Header` (0 / 3 / 7 / 11bits)：这一字段对正在发送的消息 (不管是整个消息，还是只是一小部分) 的信息进行编码。这一字段的长度可以使用块头中定义的块类型进行决定。
- `Extended Timestamp` (0 / 4 bytes)：这一字段是否出现取决于块消息头中的 `timestamp` 或者 `timestamp delta` 字段。更多信息参考

### Chunk Basic Header

`chunk Basic Header` 对 `chunk stream id` 和块类型 (由下图中的 fmt 字段表示) 进行编码。`chunk Basic Header` 字段可能会有 `1`，`2` 或者 `3` 个字节，取决于 `chunk stream id`。

一个 `RTMP` 实现应该使用能够容纳这个 `id` 的最小的容量进行表示。

`RTMP` 协议最多支持 `65597` 个流，`chunk stream id` 范围 `3 - 65599`。`id` `0`、`1`、`2` 被保留。

- `0` 值表示二字节形式，并且 `chunk stream id` 范围 `64 - 319`

         0                   1
         0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |fmt|     0     |   cs id - 64  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
              Chunk Basic Header 2

- `1` 值表示三字节形式，并且 `chunk stream id` 范围为 `64 - 65599`

         0                   1                   2
         0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |fmt|     1     |           cs id - 64          |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                    chunk Basic Header 3

- `3 - 63` 范围内的值表示整个流 `chunk stream id`

          0 1 2 3 4 5 6 7
         +-+-+-+-+-+-+-+-+
         |fmt|   cs id   |
         +-+-+-+-+-+-+-+-+
        chunk Basic Header 1

- 带有 `2` 值的 `chunk stream id` 被保留，用于下层协议控制消息和命令。

协议中字段代表的意义如下：

- `fmt` (2bits)：这一字段指示 `chunk Message Header` 使用的四种格式之一。
- `cs id` (6bits)：这一字段包含有 `chunk stream id`，值的范围是 `2 - 63`。值 `0` 或 `1` 用于指示这一字段是 2 或 3 字节版本。
- `cs id - 64` (8/16bits)：这一字段包含了 `chunk stream id` 减掉 `64` 后的值。例如，`chunk stream id` 为 `365` 时会在 `cs id` 中会以一个 `1` 和 `cs id - 64` 中的一个 `16` 位 的 `301` 进行表示。

    > `chunk stream id` `64 - 319` 可以使用 2-byte 或者 3-byte 的形式在头中表示。

### Message Header

- `Type 0`：块头的长度是 `11bits`。这一类型必须用在 `chunk stream` 的起始位置，和流 `timestamp` 重来的时候 (比如，重置)。

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                     timestamp                 |message length |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |     message length (cont)     |message type id| msg stream id |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |           message stream id (cont)            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                        chunk Message Header - Type 0

    > `timestamp` (`3bits`)：对于 type-0 块，当前消息的绝对 `timestamp` 在这里发送。如果 `timestamp` 大于或者等于 `16777215` (十六进制 `0xFFFFFF`)，这一字段必须是 `16777215`，表明有扩展 `timestamp` 字段来补充完整的 `32` 位 `timestamp`。否则的话，这一字段必须是整个的 `timestamp`。

- `Type 1`：块头长为 `7bits`。不包含消息流 `id`；这一块使用前一块一样的流 `id`。可变长度消息的流 (例如，一些视频格式) 应该在第一块之后使用这一格式表示之后的每个新消息。

         0                   1                   2                   3
         0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                timestamp delta                |message length |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |     message length (cont)     |message type id|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                          chunk Message Header - Type 1

- `Type 2`：块头长度为 `3bits`。既不包含流 `id` 也不包含消息长度；这一块具有和前一块相同的流 id 和消息长度。具有不变长度的消息 (例如，一些音频和数据格式) 应该在第一块之后使用这一格式表示之后的每个新消息。

         0                   1                   2
         0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                timestamp delta                |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                  chunk Message Header - Type 2

- `Type 3`：块没有消息头。流 id、消息长度以及 timestamp delta 等字段都不存在；这种类型的块使用前面块一样的 chunk stream id。当单一一个消息被分割为多块时，除了第一块的其他块都应该使用这种类型。参考例 2 (5.3.2.2 小节)。组成流的消息具有同样的大小，流 id 和时间间隔应该在类型 2 之后的所有块都使用这一类型。参考例 1 (5.3.2.1 小节)。如果第一个消息和第二个消息之间的 delta 和第一个消息的 timestamp 一样的话，那么在类型 0 的块之后要紧跟一个类型 3 的块，因为无需再来一个类型 2 的块来注册 delta 了。如果一个类型 3 的块跟着一个类型 0 的块，那么这个类型 3 块的 timestamp delta 和类型 0 块的 timestamp 是一样的。

块消息头中各字段的描述如下：

- `timestamp delta` (3bits)：对于一个类型 1 或者类型 2 的块，前一块的 timestamp 和当前块的 timestamp 的区别在这里发送。如果 delta 大于或者等于 16777215 (十六进制 0xFFFFFF)，那么这一字段必须是为 16777215，表示具有扩展 timestamp 字段来对整个 32 位 delta 进行编码。否则的话，这一字段应该是为具体 delta。
- `message length` (3bits)：对于一个类型 0 或者类型 1 的块，消息长度在这里进行发送。注意这通常不同于块的有效载荷的长度。块的有效载荷代表所有的除了最后一块的最大块大小，以及剩余的 (也可能是小消息的整个长度) 最后一块。
- `message type id` (1bit)：对于类型 0 或者类型 1 的块，消息的类型在这里发送。
- `message stream id` (4bits)：对于一个类型为 0 的块，保存消息流 id。消息流 id 以小端格式保存。所有同一个 chunk stream 下的消息都来自同一个消息流。当可以将不同的消息流组合进同一个 chunk stream 时，这种方法比头压缩的做法要好。但是，当一个消息流被关闭而其他的随后另一个是打开着的，就没有理由将现有 chunk stream 以发送一个新的类型 0 的块进行复用了。

### Extended Timestamp

`extended timestamp` 字段用于对大于 `16777215` (`0xFFFFFF`) 的 `timestamp` 或者 `timestamp delta` 进行编码；也就是，对于不适合于在 `24` 位的 `Type 0`、`Type 1` 和 `Type 2` 的块里的 `timestamp` 和 `timestamp delta` 编码。这一字段包含了整个 `32` 位的 `timestamp` 或者 `timestamp delta` 编码。可以通过设置 `Type 0` 块的 `timestamp` 字段、`Type 1` 或者 `Type 2` 块的 `timestamp delta` 字段 `16777215` (`0xFFFFFF`) 来启用这一字段。当最近的具有同一 `chunk stream` 的 `Type 0`、`Type 1` 和 `Type 2` 指示扩展 `timestamp` 字段出现时，这一字段才会在 `Type 3` 的块中出现。

## Chunk Data

`chunk data` 的长度可变，是当前块的有效负载。

- `chunk data` 的长度由 `chunk header` 中的 `message length` 决定
- `chunk data` 的类型由 `chunk header` 中的 `message type id` 决定

根据 message type id 的不同，可以将 `chunk data` 分为以下类型：

- 协议控制消息 （Protocol Control Messages）
- 用户控制消息 （User Control Message）
- RTMP 命令消息 （RTMP Command Messages）

### 协议控制消息 (1, 2, 3, 5, 6)

协议控制消息主要用来沟通 `RTMP` 初始状态的相关连接信息，比如：`Windows Size`，`chunk Size` 等。

协议控制消息中一共有 5 种不同的 `Message` 类型，是根据 `chunk Header` 中 `Message Header` 的 `message type id` 决定的，取值范围为 1, 2, 3, 5, 6。

另外，协议控制消息在构造的时候需要注意，其 `chunk Header` 中的 `message stream id` 和 `chunk stream id` 需要设置为固定值：

- `message stream id == 0`
- `chunk stream id == 2`

#### 设置块大小 (1)

设置块大小，被用来通知对方新的最大的块大小。

默认最大的块大小为 128 字节，客户端和服务器可以使用此消息来修改默认的块大小。例如，假设客户端想要发送的音频数据大小为 131 字节，而块大小为 128 字节。在这种情况下，客户端可以通知服务器新的块大小为 131 字节，然后就可以使用一个块来发送完整的音频数据了。

最大的块大小至少为 128 字节，块至少携带 1 个字节的内容。通信的每一个方向（例如从客户端到服务器）拥有独立的块大小设置。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |0|                     chunk size (31 bits)                    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            Payload for the 'Set Chunk Size' protocol message

- `0`：当前比特位必须为零。
- `chunk size` (31bits)：本字段标识了新的最大块大小，以字节为单位，发送端之后将使用此值作为最大的块大小。本字段的有效值为 `1-2147483647(0x7FFFFFFF)`，由于消息的最大长度为 `16777215(0xFFFFFF)`，而一个块最多只能携带一条消息，因此本字段的实际有效值为 `1-16777215(0xFFFFFF)`。

#### 中断消息 (2)

中断消息，用来通知通信的对方，如果正在等待一条消息的部分块（已经接收了一部分），那么可以丢弃
之前已经接收到的块。通信的一方将接收到 `chunk stream id` 作为当前协议消息的有效数据。应用程序可以发送此消息来通知对方，当前正在传输的消息没有必要再处理了。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       chunk stream id (32 bits)               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          Payload for the 'Abort Message' protocol message

- `chunk stream id`(32bits)：本字段中 `chunk stream id` 用来标识哪个 `chunk stream id` 的消息将被丢弃。

#### 应答 (3)

客户端和服务器在接收到与接收窗口大小相等的数据后，必须发送应答消息给对方。窗口大小的定义为发送方在接收到接收方的任何应答前，可以发送的最大数据量。本消息包含了序列号，序列号为截至目前接收到的数据总和，以字节为单位。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                        sequence number (4 bytes)              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           Payload for the 'Acknowledgement' protocol message

- 序列号 (32bits)：本字段包含了截止目前接收到的数据总和，以字节为单位。

#### 窗口确认大小 (5)

客户端和服务器发送这个消息来通知对方应答窗口的大小。发送方在发送了等于窗口大小的数据之后，等待接收对方的应答消息（在接收到应答之前停止发送数据）。接收方必须发送应答消息，在会话开始时，或从上一次发送应答之后接收到了等于窗口大小的数据。

这是用来协商发送包的大小的。这个和上面的 chunk size 不同，这里主要针对的是客户端可接受的最大数据包的值，而 chunk size 是指每次发送的包的大小。一般电脑设置的大小都是 500000B

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                   Acknowledgement Window size (4 bytes)       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     Payload for the 'Window Acknowledgement Size' protocol message

#### 设置对端带宽 (6)

客户端和服务器发送此消息来说明对方的出口带宽限制。接收方以此来限制自己的出口带宽，即限制未被应答的消息数据大小。接收到此消息的一方，如果窗口大小与上次发送的不一致，应该回复应答窗口大小的消息。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                   Acknowledgement Window size                 |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  Limit Type   |
    +-+-+-+-+-+-+-+-+
          Payload for the 'Set Peer Bandwidth' protocol message

* 限制类型的取值为下面之一：
    - 硬限制（0）：应该限制出口带宽为指明的窗口大小。
    - 软限制（1）：应该限制出口带宽为指明的窗口大小，或已经生效的小一点的窗口大小。
    - 动态限制（2）：如果上一次为硬限制，此消息被视为硬限制，否则忽略此消息。

### 用户控制消息 (4)

客户端或者服务器端发送这一消息来通知对端用户控制事件。

### 命令消息

- 命令消息 (20, 17)
- 数据消息 (18, 15)
- 共享对象消息 (19, 16)
- 音频消息 (8)
- 视频消息 (9)
- 统计消息 (22)

#### 命令消息 (20, 17)

命令消息在客户端和服务器端传递 AMF 编码的命令。这些消息被分配以消息类型值为 20 以进行 `AMF0` 编码，消息类型值为 17 以进行 `AMF3` 编码。

这些消息发送以进行一些操作，比如，连接，创建流，发布，播放，对端暂停。命令消息，像 `onstatus`、`result` 等等，用于通知发送者请求的命令的状态。一个命令消息由命令名、事务 id 和包含相关参数的命令对象组成。一个客户端或者一个服务器端可以通过和对端通信的流使用这些命令消息请求远程调用 (RPC)。

客户端和服务器通过 `AMF` 编码的数据交换命令。发送者发送包含命令名称，事务 ID，包含相关参数的命令对象的消息。例如，通过连接命令中包含的 APP 参数来告诉服务器连接的对方是哪个客户端。接收方处理命令消息，并使用相同的事务 ID 应答。应答字符串为 `_result` 或 `_error` 或方法名，例如 `verifyClient` 或 `contactExternalServer`。事务 ID 标明了应答指向的命令。事务 ID 相当于 `IMAP` 协议或其他协议中的标签。命令字符串中的方法名，表明了发送端想要在接收端执行的方法。

下面的类对象被用来发送各种命令：

`NetConnection`：服务器和客户端之间进行网络连接的一种高级表示形式。
`NetStream`：代表了发送音频流，视频流，或其他相关数据的频道。当然还有一些像播放，暂停之类的命令，用来控制数据流。

##### NetConnection

网络连接管理着客户端和服务器之间的双向连接。另外，它也支持异步远程命令调用。

网络连接允许使用以下的命令：

- 连接 connect
- 调用 call
- 停止 close
- 创建流 createStream

###### connect

客户端发送连接命令给服务器，来获取一个和服务器通信的实例。客户端发送给服务器的命令结构如下：

| Field Name | Type | Description |
| - | - | - |
| Command Name | String | Name of the command. Set to "connect". |
| Transaction ID | Number | Always set to 1. |
| Command Object | Object | Command information object which has the name-value pairs. |
| Optional User Arguments | Object  | Any optional information              |

下面是连接命令的命令对象里包含的键值对的说明：

| Property | Type | Description | Example Value  |
| - | - | - | - |
| app | String | The Server application name the client is connected to. | testapp |
| flashver | String | Flash Player version. It is the same string as returned by the ApplicationScript getversion () function. | FMSc/1.0 |
| swfUrl | String | URL of the source SWF file making the connection. | file://C:/FlvPlayer.swf |
| tcUrl | String | URL of the Server. It has the following format. protocol://servername:port/appName/appInstance | rtmp://localhost:1935/testapp/instance1 |
| fpad | Boolean| True if proxy is being used.| true or false  |
| audioCodecs | Number | Indicates what audio codecs the client supports. | SUPPORT_SND_MP3 |
| videoCodecs | Number | Indicates what video codecs are supported. | SUPPORT_VID_SORENSON |
| videoFunction | Number | Indicates what special video functions are supported. | SUPPORT_VID_CLIENT_SEEK |
| pageUrl | String | URL of the web page from where the SWF file was loaded. | <http://somehost/sample.html> |
| object Encoding | Number | AMF encoding method. | AMF3 |

音频编码属性的可选值：

原始 PCM，ADPCM，MP3，NellyMoser（5，8,11，16，22，44kHz），AAC，Speex。

| Codec Flag | Usage | Value |
|       -    |   -   |   -   |
|  SUPPORT_SND_NONE    | Raw sound, no compression  | 0x0001 |
|  SUPPORT_SND_ADPCM   | ADPCM compression | 0x0002 |
|  SUPPORT_SND_MP3     | mp3 compression | 0x0004 |
|  SUPPORT_SND_INTEL   | Not used | 0x0008 |
|  SUPPORT_SND_UNUSED  | Not used | 0x0010 |
|  SUPPORT_SND_NELLY8  | NellyMoser at 8-kHz compression | 0x0020 |
|  SUPPORT_SND_NELLY   | NellyMoser compression (5, 11, 22, and 44 kHz) | 0x0040 |
|  SUPPORT_SND_G711A   | G711A sound compression (Flash Media Server only) | 0x0080 |
|  SUPPORT_SND_G711U   | G711U sound compression (Flash Media Server only) | 0x0100 |
|  SUPPORT_SND_NELLY16 | NellyMouser at 16-kHz compression | 0x0200 |
|  SUPPORT_SND_AAC     | Advanced audio coding (AAC) codec | 0x0400 |
|  SUPPORT_SND_SPEEX   | Speex Audio | 0x0800 |
|  SUPPORT_SND_ALL     | All RTMP-supported audio codecs | 0x0FFF |

###### call

网络连接对象中包含的 call 方法，会在接收端执行远程过程调用（RPC）。被调用的 RPC 方法名作为 call 方法的参数传输。

从发送端到接收端的命令结构如下：

| Field Name | Type | Description |
|       -    |  -   |      -      |
| Procedure Name | String | Name of the remote procedure that is called.  |
| Transaction ID | Number | If a response is expected we give a transaction Id. Else we pass a value of 0 |
| Command Object | Object | If there exists any command info this is set, else this is set to null type. |
| Optional Arguments | Object | Any optional arguments to be provided |

应答的命令结构如下：

| Field Name | Type | Description |
|       -    |  -   |      -      |
| Command Name | String | Name of the command. |
| Transaction ID | Number | ID of the command, to which the response belongs. |
| Command Object | Object  | If there exists any command info this is set, else this is set to null type. |
| Response | Object | Response from the method that was called. |

###### createStream

客户端通过发送此消息给服务器来创建一个用于消息交互的逻辑通道。音频，视频，和元数据都是通过 createStream 命令创建
的流通道发布出去的。

NetConnection 是默认的交互通道，流 ID 为0. 协议和一部分命令消息，包含 createStream，都是使用默认的交互通道发布
的。

从客户端发送给服务器的命令结构如下：

| Field Name | Type | Description |
|       -    |  -   |      -      |
| Command Name | String | Name of the command. Set to "createStream". |
| Transaction ID | Number | Transaction ID of the command. |
| Command Object | Object | If there exists any command info this is set, else this is set to null type. |

从服务器发送给客户端的命令结构：

| Field Name | Type | Description |
|       -    |  -   |      -      |
| Command Name | String | \_result or \_error; indicates whether the response is result or error. |
| Transaction ID | Number | ID of the command that response belongs to. |
| Command Object | Object | If there exists any command info this is set, else this is set to null type. |
| Stream ID | Number | The return value is either a stream ID or an error information object. |

##### NetStream

网络流定义了通过网络连接把音频，视频和数据消息流在客户端和服务器之间进行交换的通道。一个网络连接对象可以有多个
网络流，进而支持多个数据流。

客户端可以通过网络流发送到服务器的命令如下：

- 播放play
- 播放2 play2
- 删除流 deleteStream
- 关闭流 closeStream
- 接收音频 receiveAudio
- 接收视频 receiveVideo
- 发布 publish
- 定位 seek
- 暂停 pause

服务器通过发送 onStatus 命令给客户端来通知网络流状态的更新。

    +--------------+----------+----------------------------------------+
    | Field Name   |   Type   |             Description                |
    +--------------+----------+----------------------------------------+
    | Command Name |  String  | The command name "onStatus".           |
    +--------------+----------+----------------------------------------+
    | Transaction  |  Number  | Transaction ID set to 0.               |
    | ID           |          |                                        |
    +--------------+----------+----------------------------------------+
    | Command      |  Null    | There is no command object for         |
    | Object       |          | onStatus messages.                     |
    +--------------+----------+----------------------------------------+
    | Info Object  | Object   | An AMF object having at least the      |
    |              |          | following three properties: "level"    |
    |              |          | (String): the level for this message,  |
    |              |          | one of "warning", "status", or "error";|
    |              |          | "code" (String): the message code, for |
    |              |          | example "NetStream.Play.Start"; and    |
    |              |          | "description" (String): a human-       |
    |              |          | readable description of the message.   |
    |              |          | The Info object MAY contain other      |
    |              |          | properties as appropriate to the code. |
    +--------------+----------+----------------------------------------+
               Format of NetStream status message commands.

#### 数据消息 (18, 15)

客户端或者服务器端通过发送这些消息以发送元数据或者任何用户数据到对端。元数据包括数据 (音频，视频等等) 的详细信息，比如创建时间，时长，主题等等。这些消息被分配以消息类型为 18 以进行 AMF0 编码和消息类型 15 以进行 AMF3 编码。

#### 共享对象消息 (19, 16)

所谓共享对象其实是一个 Flash 对象 (一个名值对的集合)，这个对象在多个不同客户端、应用实例中保持同步。消息类型 19 用于 AMF0 编码、16 用于 AMF3 编码都被为共享对象事件保留。每个消息可以包含有不同事件。

#### 音频消息 (8)

客户端或者服务器端发送这一消息以发送音频数据到对端。消息类型 8 为音频消息保留。

#### 视频消息 (9)

客户端或者服务器发送这一消息以发送视频数据到对端。消息类型 9 为视频消息保留。

#### 统计消息 (22)

统计消息是一个单一的包含一系列的使用 6.1 节描述的 RTMP 子消息的消息。消息类型 22 用于统计消息。

## 参考资料

* [1] [《Adobe’s Real Time Messaging Protocol》](https://wwwimages2.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
