---
title: 源码阅读 rtmpdump - 主函数解析
categories:
  - 源码阅读
tags:
  - rtmpdump
abbrlink: e7a6677c
date: 2020-10-19 15:29:10
---
`main` 函数位于源码文件 `rtmpdump.c` 中，是 `rtmpdump` 工具的入口函数。

<!--more-->

## main 函数源码

由于 `main` 函数较长，本文对其中部分内容进行了裁剪，比如以下不会走到的 `if`、`case` 分支等。

假设我们执行的命令为：`rtmpdump -r rtmp://62.234.111.13:1935/mylive/1 -o Hello.flv`，该条命令涉及到的函数流程均进行保留，其他流程分支请参考源代码。

``` c++
int main(int argc, char **argv) {
    if (!InitSockets()) {
        RTMP_Log(RTMP_LOGERROR,
        "Couldn't load sockets support on your platform, exiting!");
        return RD_FAILED;
    }

    RTMP_Init(&rtmp);

    while ((opt = getopt_long(argc, argv,
                "hVveqzRr:s:t:i:p:a:b:f:o:u:C:n:c:l:y:Ym:k:d:A:B:T:w:x:W:X:S:#j:",
                longopts, NULL)) != -1) {
        switch (opt) {
            case 'i':
                STR2AVAL(fullUrl, optarg);
                break;
            case 'o':
                flvFile = optarg;
                if (strcmp(flvFile, "-")) bStdoutMode = FALSE;
                break;
            default:
                RTMP_LogPrintf("unknown option: %c\n", opt);
                usage(argv[0]);
                return RD_FAILED;
                break;
        }
    }

    if (port == 0 && !fullUrl.av_len) {
        if (protocol & RTMP_FEATURE_SSL) port = 443;
        else if (protocol & RTMP_FEATURE_HTTP) port = 80;
        else port = 1935;
    }

    if (tcUrl.av_len == 0)
    {
        tcUrl.av_len = strlen(RTMPProtocolStringsLower[protocol]) +
            hostname.av_len + app.av_len + sizeof("://:65535/");
        tcUrl.av_val = (char *) malloc(tcUrl.av_len);
        if (!tcUrl.av_val) return RD_FAILED;
        tcUrl.av_len = snprintf(tcUrl.av_val, tcUrl.av_len, "%s://%.*s:%d/%.*s",
            RTMPProtocolStringsLower[protocol], hostname.av_len,
            hostname.av_val, port, app.av_len, app.av_val);
    }

    if (!fullUrl.av_len) {
        RTMP_SetupStream(&rtmp, protocol, &hostname, port, &sockshost, &playpath,
               &tcUrl, &swfUrl, &pageUrl, &app, &auth, &swfHash, swfSize,
               &flashVer, &subscribepath, &usherToken, dSeek, dStopOffset, bLiveStream, timeout);
    }
    else {
        if (RTMP_SetupURL(&rtmp, fullUrl.av_val) == FALSE) {
          RTMP_Log(RTMP_LOGERROR, "Couldn't parse URL: %s", fullUrl.av_val);
          return RD_FAILED;
        }
    }

    if (!file) {
        if (bStdoutMode) {
            file = stdout;
            SET_BINMODE(file);
        }
        else {
            file = fopen(flvFile, "w+b");
            if (file == 0) {
                RTMP_LogPrintf("Failed to open file! %s\n", flvFile);
                return RD_FAILED;
            }
        }
    }

    int first = 1;
    while (!RTMP_ctrlC) {
        RTMP_Log(RTMP_LOGDEBUG, "Setting buffer time to: %dms", bufferTime);
        RTMP_SetBufferMS(&rtmp, bufferTime);

        if (first) {
            first = 0;
            if (!RTMP_Connect(&rtmp, NULL)) {
                nStatus = RD_NO_CONNECT;
                break;
            }
            // User defined seek offset
            if (dStartOffset > 0) {
                // Don't need the start offset if resuming an existing file
                if (bResume) {
                    RTMP_Log(RTMP_LOGWARNING, "Can't seek a resumed stream, ignoring --start option");
                    dStartOffset = 0;
                }
                else {
                    dSeek = dStartOffset;
                }
            }

            // Calculate the length of the stream to still play
            if (dStopOffset > 0) {
                // Quit if start seek is past required stop offset
                if (dStopOffset <= dSeek) {
                    RTMP_LogPrintf("Already Completed\n");
                    nStatus = RD_SUCCESS;
                    break;
                }
            }

            if (!RTMP_ConnectStream(&rtmp, dSeek)) {
                nStatus = RD_FAILED;
                break;
            }
        }
        else {
            nInitialFrameSize = 0;

            if (retries) {
                RTMP_Log(RTMP_LOGERROR, "Failed to resume the stream\n\n");
                if (!RTMP_IsTimedout(&rtmp)) nStatus = RD_FAILED;
                else nStatus = RD_INCOMPLETE;
                break;
            }
            RTMP_Log(RTMP_LOGINFO, "Connection timed out, trying to resume.\n\n");
            /* Did we already try pausing, and it still didn't work? */
            if (rtmp.m_pausing == 3) {
            /* Only one try at reconnecting... */
                retries = 1;
                dSeek = rtmp.m_pauseStamp;
                if (dStopOffset > 0) {
                    if (dStopOffset <= dSeek) {
                        RTMP_LogPrintf("Already Completed\n");
                        nStatus = RD_SUCCESS;
                        break;
                    }
                }
                if (!RTMP_ReconnectStream(&rtmp, dSeek)) {
                    RTMP_Log(RTMP_LOGERROR, "Failed to resume the stream\n\n");
                    if (!RTMP_IsTimedout(&rtmp)) nStatus = RD_FAILED;
                    else nStatus = RD_INCOMPLETE;
                    break;
                }
            }
            else if (!RTMP_ToggleStream(&rtmp)) {
                RTMP_Log(RTMP_LOGERROR, "Failed to resume the stream\n\n");
                if (!RTMP_IsTimedout(&rtmp)) nStatus = RD_FAILED;
                else nStatus = RD_INCOMPLETE;
                break;
            }
            bResume = TRUE;
        }

        nStatus = Download(&rtmp, file, dSeek, dStopOffset, duration, bResume,
                metaHeader, nMetaHeaderSize, initialFrame,
                initialFrameType, nInitialFrameSize, nSkipKeyFrames,
                bStdoutMode, bLiveStream, bRealtimeStream, bHashes,
                bOverrideBufferTime, bufferTime, &percent);
        free(initialFrame);
        initialFrame = NULL;

        /* If we succeeded, we're done. */
        if (nStatus != RD_INCOMPLETE || !RTMP_IsTimedout(&rtmp) || bLiveStream)
            break;
    }

    if (nStatus == RD_SUCCESS)
        RTMP_LogPrintf("Download complete\n");
    else if (nStatus == RD_INCOMPLETE)
        RTMP_LogPrintf("Download may be incomplete (downloaded about %.2f%%), try resuming\n", percent);

clean:
    RTMP_Log(RTMP_LOGDEBUG, "Closing connection.\n");
    RTMP_Close(&rtmp);
    if (file != 0) fclose(file);
    CleanupSockets();
    return nStatus;
}
```

## main 函数流程

从上述源码中可以看出，`main` 函数的大致流程为：

1. InitSockets();
2. RTMP_Init();
3. RTMP_SetupStream()/RTMP_SetupURL();
4. fopen();
5. RTMP_Connect();
6. RTMP_ConnectStream()
7. Download();
8. RTMP_Close();
9. fclose();
10. CleanupSockets();

其中 `InitSockets` 函数和 `CleanupSockets` 较为简单，在 `Windows` 平台下仅仅做了 `socket` 的初始化和反初始化工作，在 `Linux` 平台下甚至什么都没有做就 `return` 了：

``` c++
int InitSockets() {
#ifdef WIN32
    WORD version;
    WSADATA wsaData;
    version = MAKEWORD(1, 1);
    return (WSAStartup(version, &wsaData) == 0);
#else
    return TRUE;
#endif
}

inline void CleanupSockets() {
#ifdef WIN32
    WSACleanup();
#endif
}
```

## main 函数关键函数

由 main 函数流程可知，以下函数是理解 RTMPDump 源码的关键：

1. RTMP_Init();
2. RTMP_SetupStream()/RTMP_SetupURL();
3. RTMP_Connect();
4. RTMP_ConnectStream()
5. Download();
6. RTMP_Close();

由于每个函数进行分析所占用篇幅较大，以上函数的解析会在后续文章中推出，欢迎持续关注本博客。

## 参考资料

* [1] [RTMPdump 源代码分析 1： main()函数_雷霄骅(leixiaohua1020)的专栏-CSDN博客](https://blog.csdn.net/leixiaohua1020/article/details/12952977)
