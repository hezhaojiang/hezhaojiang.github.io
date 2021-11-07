---
title: 流媒体传输 - RTCP 协议
categories:
    - 流媒体传输
tags:
  - RTCP
abbrlink: 634443fb
date: 2020-07-28 23:06:30
---
## RTCP 协议

### SR (Sender Report RTCP Packet)

            0                   1                   2                   3
            0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    header |V=2|P|    RC   |   PT=SR=200   |             length            |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                         SSRC of sender                        |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    sender |              NTP timestamp, most significant word             |
    info   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |             NTP timestamp, least significant word             |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                         RTP timestamp                         |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                     sender's packet count                     |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                      sender's octet count                     |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    report |                 SSRC_1 (SSRC of first source)                 |
    block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      1    | fraction lost |       cumulative number of packets lost       |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |           extended highest sequence number received           |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                      interarrival jitter                      |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                         last SR (LSR)                         |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                   delay since last SR (DLSR)                  |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    report |                 SSRC_2 (SSRC of second source)                |
    block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      2    :                               ...                             :
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
           |                  profile-specific extensions                  |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

<!-- more -->

> [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt) : 6.4.1 SR: Sender Report RTCP Packet

### RR: Receiver Report RTCP Packet

            0                   1                   2                   3
            0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    header |V=2|P|    RC   |   PT=RR=201   |             length            |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                     SSRC of packet sender                     |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    report |                 SSRC_1 (SSRC of first source)                 |
    block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      1    | fraction lost |       cumulative number of packets lost       |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |           extended highest sequence number received           |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                      interarrival jitter                      |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                         last SR (LSR)                         |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                   delay since last SR (DLSR)                  |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    report |                 SSRC_2 (SSRC of second source)                |
    block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      2    :                               ...                             :
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
           |                  profile-specific extensions                  |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

> [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt) : 6.4.2 RR: Receiver Report RTCP Packet

### SDES: Source Description RTCP Packet

            0                   1                   2                   3
            0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    header |V=2|P|    SC   |  PT=SDES=202  |             length            |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    chunk  |                          SSRC/CSRC_1                          |
      1    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                           SDES items                          |
           |                              ...                              |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    chunk  |                          SSRC/CSRC_2                          |
      2    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                           SDES items                          |
           |                              ...                              |
           +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

> [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt) : 6.5 SDES: Source Description RTCP Packet

## 参考资料

* [1] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
* [2] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
* [3] [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt)
* [4] [RFC 6184: RTP Payload Format for H.264 Video](https://www.rfc-editor.org/rfc/rfc6184.txt)
* [5] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
