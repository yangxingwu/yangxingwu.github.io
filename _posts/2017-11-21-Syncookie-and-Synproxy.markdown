---
layout: post
title:  "Syncookie and synproxy"
date:   2017-11-21 23:00:00 +0800
categories: DDOS
---

## TCP建连过程

除了熟悉的三次握手，TCP建连过程还需要做到：  

1. Sequence synchronization  
2. Parameter exchange  
    1. MSS
    2. Window Scale factor
    3. SACK
    4. Alternate Checksum method (这个不常用)

Refer to [TCP/IP guide](http://www.tcpipguide.com/free/t_TCPConnectionEstablishmentSequenceNumberSynchroniz-3.htm).  

Syncookie以及Synproxy的难点就在于，如何在cookie中内嵌这些信息  

## Syncookie

Refer to [wiki](https://en.wikipedia.org/wiki/SYN_cookies)

### 什么是syncookie

> SYN cookie is a technique used to resist SYN flood attacks. The technique's primary inventor Daniel J. Bernstein defines SYN cookies as "**particular choices of initial TCP sequence numbers by TCP servers**." In particular, the use of SYN cookies allows a server to avoid dropping connections when the SYN queue fills up. Instead, the server behaves as if the SYN queue had been enlarged. The server sends back the appropriate SYN+ACK response to the client but discards the SYN queue entry. If the server then receives a subsequent ACK response from the client, the server is able to reconstruct the SYN queue entry using information encoded in the TCP sequence number.

简单的来讲，syncookie就是在收到客户端的syn包，回复syn+ack时，选择一个特殊的sequence；其中sequence中内嵌了客户端第一次syn包中所带的部分信息（如MSS），以便在三次握手完成时重建SYN queue entry。  

### 实现

In order to initiate a TCP connection, the client sends a TCP SYN packet to the server. In response, the server sends a TCP SYN+ACK packet back to the client. One of the values in this packet is a sequence number, which is used by the TCP to reassemble the data stream. According to the TCP specification, that first sequence number sent by an endpoint can be any value as decided by that endpoint. **SYN cookies are initial sequence numbers that are carefully constructed according to the following rules**:

- let t be a slowly incrementing timestamp (typically time() logically right-shifted 6 positions, which gives a resolution of 64 seconds)
- let m be the maximum segment size (MSS) value that the server would have stored in the SYN queue entry
- let s be the result of a cryptographic hash function computed over the server IP address and port number, the client IP address and port number, and the value t. The returned value s must be a 24-bit value.

The initial TCP sequence number, i.e. the SYN cookie, is computed as follows:

Top 5 bits: t mod 32  
Middle 3 bits: an encoded value representing m  
Bottom 24 bits: s  

### 缺陷

从上文中可以看到，syncookie实现的最大缺陷是，其只能在sequence中内嵌客户端SYN包中MSS信息，从而丢失其他的如timestamp、large window、SACK等信息；

> The use of SYN cookies does not break any protocol specifications, and therefore should be compatible with all TCP implementations. There are, however, two caveats that take effect when SYN cookies are in use.
> 1. Firstly, the server is limited to only 8 unique MSS values, as that is all that can be encoded in 3 bits.  
> 2. Secondly, the server must reject all TCP options (such as large windows or timestamps), because the server discards the SYN queue entry where that information would otherwise be stored.

linux kernel有关syncookie的改进：[Improving syncookies](https://lwn.net/Articles/277146/)  

## Synproxy

Synproxy，顾名思义Syn代理；也就是说中间设备先伪造Server与Client建立连接，然后再与Server建立连接，双向连接建立好之后再进行数据转发；  

![synproxy](https://www.safaribooksonline.com/library/view/junos-security/9781449381721/httpatomoreillycomsourceoreillyimages668207.png)  

其流程可大概分为以下几个阶段：

1. T1: Client与Proxy三次握手
2. T2: Proxy与server三次握手
3. T3: Client与Server间传输数据
4. T4: Proxy清理资源

其实现过程中可能遇到的挑战：

1. T1阶段 WinSize 处理
2. T1阶段其他扩展，比如TFO(TCP Fast Open)、带payload
3. T1阶段 TCP Options 的处理，包括MSS、WinScale、Timestamp等，发起前并未知 Server 配置
4. T1之后，T2还没完成之前，Client push数据会被丢弃
5. T2阶段的处理，目标是否可达、目标的端口是否开启
6. T2阶段记录Proxy生成的SeqNumber与Server生成的SeqNumber之间的差别，用于T3转发的矫正
7. T3阶段对 SeqNumber 的矫正
8. T4阶段的资源回收

### T1阶段处理

#### WinSize处理

WinSize的处理比较简单，有两种解决方案，一种取 Linux 默认大小4096，另外一种取0。当WinSize取0时，根据TCP协议，Client是不应该PUSH数据过来的，这时我们可以等T2阶段Server响应Proxy的SynAck，从中取得WinSize再通过Ack包发送给Client，同时这样也可以避免在T2握手完成之前Client Push数据过来。

#### TCP option处理

LVS中的方式是扩展syncookie，让其携带更多信息：

```c
/*
 * Generate a syncookie for ip_vs module.
 * Besides mss, we store additional tcp options in cookie "data".
 *
 * Cookie "data" format:
 * |[21][20][19-16][15-0]|
 * [21] SACKOK
 * [20] TimeStampOK
 * [19-16] snd_wscale
 * [15-12] MSSIND
 */
```

#### T1阶段防御

收到Client的Syn包，计算Syn Cookie，配置WinSize=0以及TCP Options，响应SynAck。对Client发过来的Ack包进行解析，判定是否合法。这个阶段可以丢弃同一链路上所有的push包


### T2阶段

T2阶段发生在Client全连接建立之后，当收到Client的Ack包时，我们认定该Client是合法Client（非Syn Flood 攻击源），那么我们就可以向后端发起三次握手，握手信息从T1阶段最后的Ack包中复原Client的Sequence Number， 以及SACK、WinScale、MSS等信息拼接 TCP Options，向Server发送Syn包  

当接收到Server的SynAck包时，除了给后端发送Ack完成三次握手，还需要获取其中的WinSize和Sequence Number，**将WinSize值以Ack包发给Client告诉Client可以发送数据了**（Ack包的信息，Sequence Number来源T1阶段的保存，Ack Number可以从SynAck包复原，WinSize值从SynAck包复原）  

这个阶段需要记录Proxy 生成的Sequence Number和 Sever生成的 Sequence Number 之间的差值 DeltaSeq  

这个时候后端Server有三种可能的状态，一个是主机不在线，第二个是主机在线但繁忙，第三是主机可达但是端口没开。前两种情况的表现比较一致，需要Proxy有重传机制。如果主机不在线，这个时候，Client可能会发送Window Probe过来，这时proxy也需要回复ZeroWindow（亦可丢弃，因为client会重传）。第三种情况主机返回RST，此时Proxy需要给Client发送RST并进入T4阶段。如果这个阶段刚好收到Client的 RST 或者 FIN 包时，则给后端发送RST并进入T4阶段。


### T3&T4阶段

T3阶段是数据转发阶段，需要利用T2阶段获取的DeltaSeq对双向数据包进行 Sequence Number 和 Acknowledge Number进行修正。如果收到任何一方的Reset或者FinAck包时，需要进行转发并进入T4阶段。在断开连接有四次挥手的状态变化，但这里简化处理并没有加以判断  

T4阶段一般来说比较短暂，主要是重置标志位、移除Hash key等  

## 参考资料

1. <https://en.wikipedia.org/wiki/SYN_cookies>
2. <https://en.wikipedia.org/wiki/Maximum_segment_size>
3. <http://cr.yp.to/syncookies.html>
4. <https://lwn.net/Articles/277146/>
5. <https://lwn.net/Articles/563151/>
6. <http://www.tcpipguide.com/free/t_TCPConnectionEstablishmentSequenceNumberSynchroniz-3.htm>
7. <http://people.netfilter.org/hawk/presentations/devconf2014/iptables-ddos-mitigation_JesperBrouer.pdf>
8. <http://blog.csdn.net/bigtree_3721/article/details/77619877>
9. <http://www.nosa.me/tag/tcp/>
