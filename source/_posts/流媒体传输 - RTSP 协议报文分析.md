---
title: 流媒体传输 - RTSP 协议报文分析
categories: 
    - 流媒体传输
tags:
  - RTSP
abbrlink: c8d9fa71
date: 2020-08-29 15:51:47
---
在 {% post_link "流媒体传输 - RTSP 协议" %} 中，我们分析 RTSP 协议交互的整个流程，在本篇文章中，我们对交互中携带的报文进行详细分析。

## Request

    Request     =       Request-Line              ; Section 6.1
                *(      general-header            ; Section 5
                |       request-header            ; Section 6.2
                |       entity-header )           ; Section 8.1
                        CRLF
                        [message-body]            ; Section 4.3

<!-- more -->

### Request-Line

    Request-Line = Method SP Request-URI SP RTSP-Version CRLF

    Method      =         "DESCRIBE"              ; Section 10.2
                |         "ANNOUNCE"              ; Section 10.3
                |         "GET_PARAMETER"         ; Section 10.8
                |         "OPTIONS"               ; Section 10.1
                |         "PAUSE"                 ; Section 10.6
                |         "PLAY"                  ; Section 10.5
                |         "RECORD"                ; Section 10.11
                |         "REDIRECT"              ; Section 10.10
                |         "SETUP"                 ; Section 10.4
                |         "SET_PARAMETER"         ; Section 10.9
                |         "TEARDOWN"              ; Section 10.7
                |         extension-method

    SP = space

    extension-method = token

    Request-URI = "*" | absolute_URI

    RTSP-Version = "RTSP" "/" 1*DIGIT "." 1*DIGIT

> 注意，与 HTTP / 1.1 相比，RTSP 请求始终包含绝对 URL（即，包括 方案、主机、端口），而不仅仅是绝对路径。
> Request-URI 中的星号 "*" 表示该请求不适用于特定资源，而是适用于服务器本身，并且仅在所使用的方法不一定适用于资源时才被允许。比如：
>> `OPTIONS * RTSP/1.0`

### Request Header Fields

    request-header  =      Accept                   ; Section 12.1
                    |      Accept-Encoding          ; Section 12.2
                    |      Accept-Language          ; Section 12.3
                    |      Authorization            ; Section 12.5
                    |      From                     ; Section 12.20
                    |      If-Modified-Since        ; Section 12.23
                    |      Range                    ; Section 12.29
                    |      Referer                  ; Section 12.30
                    |      User-Agent               ; Section 12.41

## Response

在接收并解释了请求消息后，接收者以 RTSP 响应消息进行响应。

响应消息除了将 HTTP 版本替换为 RTSP 版本之外，其他都适用。 此外，RTSP 定义了其他状态代码，并且没有定义某些 HTTP 代码。 表 1 中定义了有效的响应代码及其可使用的方法。

    Response    =     Status-Line         ; Section 7.1
                *(    general-header      ; Section 5
                |     response-header     ; Section 7.1.2
                |     entity-header )     ; Section 8.1
                      CRLF
                      [message-body]    ; Section 4.3

### Status-Line

响应消息的第一行是状态行，由协议版本，后跟数字状态代码以及与状态代码关联的文本短语组成，每个元素由 SP 字符分隔。 除最后的 CRLF 序列外，不允许有 CR 或 LF。

    Status-Line    =    RTSP-Version SP Status-Code SP Reason-Phrase CRLF

    Status-code    =    3DIGIT

    Reason-Phrase  =    *<TEXT, excluding CR, LF>

#### Status Code 和 Reason-Phrase

Status Code 是 3 位数的整数结果代码，表示请求是否被理解或被满足。这些代码在第 11 节中有完整定义。Reason-Phrase 旨在简要说明 Status Code。Status Code 供机器使用，Reason-Phrase 供人类用户使用。 客户端不需要检查或显示 Reason-Phrase。

Status Code 的第一位数字定义了回应的类别，后面两位数字没有具体分类。首位数字有 5 种取值可能：

- 1xx：保留，将来使用。
- 2xx：成功：操作被接收、理解、接受（received, understood, accepted）。
- 3xx：重定向（Redirection）：要完成请求必须进行进一步操作。
- 4xx：客户端出错：请求有语法错误或无法实现。
- 5xx：服务器端出错：服务器无法实现合法的请求。

> 具体 Status Code 含义可查看 RFC 2326 Session 11 : Status Code Definitions

### Response Header Fields

响应头字段允许请求接收者传递关于响应的其他信息，这些信息不能放在状态行中。 这些头字段提供有关服务器以及对由 Request-URI 标识的资源的进一步访问的信息。

    response-header  =     Location             ; Section 12.25
                     |     Proxy-Authenticate   ; Section 12.26
                     |     Public               ; Section 12.28
                     |     Retry-After          ; Section 12.31
                     |     Server               ; Section 12.36
                     |     Vary                 ; Section 12.42
                     |     WWW-Authenticate     ; Section 12.44

## Method Definitions

### OPTIONS

OPTIONS 用于询问 Server 有哪些方法可用

    C->S:  OPTIONS * RTSP/1.0
           CSeq: 1
           Require: implicit-play
           Proxy-Require: gzipped-messages

    S->C:  RTSP/1.0 200 OK
           CSeq: 1
           Public: DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE

### DESCRIBE

DESCRIBE 方法从服务器检索表示的描述或媒体对象，这些资源通过请求统一资源定位符（the request URL）识别。此方法可能结合使用 Accept 首部域来指定客户端理解的描述格式。服务器端用被请求资源的描述对客户端作出响应。 DESCRIBE 的答复 - 响应对（reply-response pair）组成了 RTSP 的媒体初始化阶段。

    C->S: DESCRIBE rtsp://server.example.com/fizzle/foo RTSP/1.0
          CSeq: 312
          Accept: application/sdp, application/rtsl, application/mheg

    S->C: RTSP/1.0 200 OK
          CSeq: 312
          Date: 23 Jan 1997 15:35:06 GMT
          Content-Type: application/sdp
          Content-Length: 376

          v=0
          o=mhandley 2890844526 2890842807 IN IP4 126.16.64.4
          s=SDP Seminar
          i=A Seminar on the session description protocol
          u=http://www.cs.ucl.ac.uk/staff/M.Handley/sdp.03.ps
          e=mjh@isi.edu (Mark Handley)
          c=IN IP4 224.2.17.12/127
          t=2873397496 2873404696
          a=recvonly
          m=audio 3456 RTP/AVP 0
          m=video 2232 RTP/AVP 31
          m=whiteboard 32416 UDP WB
          a=orient:portrait

DESCRIBE 响应必须包含它描述的资源的所有媒体初始化信息。 如果媒体客户端从 DESCRIBE 以外的来源获得媒体描述，并且该描述包含完整的媒体初始化参数集，则客户端应使用这些参数，而不是通过 RTSP 请求相同媒体描述。

DESCRIBE 响应携带的 SDP 信息详见：

> {% post_link "流媒体传输 - SDP 协议" %}

### SETUP

    C->S: SETUP rtsp://example.com/foo/bar/baz.rm RTSP/1.0
          CSeq: 302
          Transport: RTP/AVP;unicast;client_port=4588-4589

    S->C: RTSP/1.0 200 OK
          CSeq: 302
          Date: 23 Jan 1997 15:35:06 GMT
          Session: 47112344
          Transport: RTP/AVP;unicast;client_port=4588-4589;server_port=6256-6257

URI 的 SETUP 请求指定要用于流媒体的传输机制。客户端可以对已经播放的流发出 SETUP 请求，以更改传输参数（服务器可以允许）。如果不允许这样做，它必须以错误 "455 方法在此状态下无效" 响应。为了使任何中间的防火墙受益，客户端必须指示传输参数，即使它对这些参数没有影响，例如，在服务器通告固定的多播地址的地方。

由于 SETUP 包含所有传输初始化信息，因此防火墙和其他中间网络设备（需要此信息）可以省去解析 DESCRIBE 响应的更艰巨的任务，该响应已保留给媒体初始化。

Transport Header 指定客户端可接受的数据传输传输参数；响应将包含服务器选择的传输参数。

#### Transport

以下是与 Transport 关联的配置参数：

    Transport           =    "Transport" ":" SP transport-spec
    transport-spec      =    transport-protocol/profile[/lower-transport]
                             *parameter
    transport-protocol  =    "RTP"
    profile             =    "AVP"
    lower-transport     =    "TCP" | "UDP"
    parameter           =    ("unicast" | "multicast")
                        |    ";" "destination" ["=" address]
                        |    ";" "interleaved" "=" channel ["-" channel]
                        |    ";" "append"
                        |    ";" "ttl" "=" ttl
                        |    ";" "layers" "=" 1*DIGIT
                        |    ";" "port" "=" port ["-" port]
                        |    ";" "client_port" "=" port ["-" port]
                        |    ";" "server_port" "=" port ["-" port]
                        |    ";" "ssrc" "=" ssrc
                        |    ";" "mode" = <"> 1\#mode <">
    ttl                 =    1*3(DIGIT)
    port                =    1*5(DIGIT)
    ssrc                =    8*8(HEX)
    channel             =    1*3(DIGIT)
    address             =    host
    mode                =    <"> *Method <"> | Method

- unicast | multicast:

    二选一地指定是进行单播还是多播的传输尝试。 默认值为多播。 单播和多播传输均能处理的客户端必须通过包括两个完整的传输规范以及每个独立的参数来指示这种能力

- destination:
    流将被发送到的地址。 客户端可以使用该参数指定多播地址。为了避免在不知情的情况下被用于拒绝服务攻击，服务器应该对客户端进行身份验证，并应该记录此类尝试，然后再允许客户端将媒体流定向到服务器未选择的地址。 这对通过 UDP 发出的 RTSP 命令尤其重要，但是实现不能依赖 TCP 作为可靠的客户端验证手段。 服务器应该不允许客户端将媒体流定向到与命令来源不同的地址上去

- mode
    mode指示会话支持的方法。有效的值有 PLAY 和 RECORD。如果没有提供，默认值是 PLAY

- port
    该参数为多播会话提供 RTP/RTCP 端口号对，它以范围的方式给出，例如：port=34567-34568

- client_port
    该参数提供客户端选择的接收媒体数据和控制信息的单播 RTP/RTCP 端口号对，它以范围的方式给出，例如：client_port=34567-34568

- server_port
    该参数提供服务端选择的接收媒体数据和控制信息的单播 RTP/RTCP 端口号对，它以范围的方式给出，例如：server_port=45678-45679

- ssrc
    该参数指示媒体服务器应该用的（请求）或将要用的（响应）RTSP SSRC 值，该参数只对单播传输有效，他是和媒体流关联的同步源的标识

例如：

Transport: RTP/AVP;multicast;ttl=127;mode="PLAY",
           RTP/AVP;unicast;client_port=3456-3457;mode="PLAY"

### PLAY

PLAY 方法告诉服务器通过 SETUP 规定的机制开始传输数据。客户端不允许在 SETUP 请求在明确确认成功以前发送 PLAY 请求。

PLAY 请求通过给出相对于正常播放时间的开始时间和结束时间，来给出播放范围。从开始时间开始传输流数据直到到达结束时间。PLAY 可以用管道（队列）形式，服务器必须按顺序执行收到的 PLAY 请求。即：在前一个 PLAY 请求依然活动期间收到 PLAY 请求，要延迟到前一个 PLAY 请求执行完毕后开始生效。

例如：不管下面例子中的两个 PLAY 请求的到达间隔多短，服务器将先从第 10 秒播放至第 15 秒，然后马上播放第 20 秒到第 25 秒，再从第 30 秒播放至结束。

    C->S: PLAY rtsp://audio.example.com/audio RTSP/1.0
          CSeq: 835
          Session: 12345678
          Range: npt=10-15
    C->S: PLAY rtsp://audio.example.com/audio RTSP/1.0
          CSeq: 836
          Session: 12345678
          Range: npt=20-25
    C->S: PLAY rtsp://audio.example.com/audio RTSP/1.0
          CSeq: 837
          Session: 12345678
          Range: npt=30-

### TEARDOWN

TEARDOWN 请求会停止所给 URI 的流传输，释放与它相关的资源。如果所给的 URI 是这个表示的表示 URI，任何与此会话相关的会话标识都将不再有效。除非所有传输参数都在会话描述中定义了，否则在再次播放前必须发送 SETUP 请求。

## 参考资料

* [1] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
