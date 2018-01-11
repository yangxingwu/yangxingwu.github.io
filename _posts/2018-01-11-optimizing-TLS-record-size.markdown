---
layout: post
title:  "Optimizing TLS record size"
date:   2018-01-11 17:25:00 +0800
categories: nginx
---

第一个问题，什么是`SSL/TLS record size`？

首先看看`SSL_read`，参见[openssl wiki](https://wiki.openssl.org/index.php/Manual:SSL_read(3))  

```c
#include <openssl/ssl.h>
int SSL_read(SSL *ssl, void *buf, int num);
```

> `SSL_read()` tries to read num bytes from the specified ssl into the buffer buf.
>
> NOTES
> If necessary, SSL_read() will negotiate a TLS/SSL session, if not already explicitly performed by SSL_connect(3) or SSL_accept(3). If the peer requests a re-negotiation, it will be performed transparently during the SSL_read() operation. The behaviour of SSL_read() depends on the underlying BIO.
> 
> For the transparent negotiation to succeed, the ssl must have been initialized to client or server mode. This is being done by calling SSL_set_connect_state(3) or SSL_set_accept_state() before the first call to an SSL_read() or SSL_write(3) function.
> 
> SSL_read() ==works based on the SSL/TLS records==. The data are received in records (with a maximum record size of 16kB for SSLv3/TLSv1). ==Only when a record has been completely received, it can be processed (decryption and check of integrity)==. Therefore data that was not retrieved at the last call of SSL_read() can still be buffered inside the SSL layer and will be retrieved on the next call to SSL_read(). If num is higher than the number of bytes buffered, SSL_read() will return with the bytes buffered. If no more bytes are in the buffer, SSL_read() will trigger the processing of the next record. Only when the record has been received and processed completely, SSL_read() will return reporting success. At most the contents of the record will be returned. ==As the size of an SSL/TLS record may exceed the maximum packet size of the underlying transport (e.g. TCP), it may be necessary to read several packets from the transport layer before the record is complete and SSL_read() can succeed==.
> 
> If the underlying BIO is blocking, SSL_read() will only return, once the read operation has been finished or an error occurred, except when a renegotiation take place, in which case a SSL_ERROR_WANT_READ may occur. This behaviour can be controlled with the SSL_MODE_AUTO_RETRY flag of the SSL_CTX_set_mode(3) call.
> 
> If the underlying BIO is non-blocking, SSL_read() will also return when the underlying BIO could not satisfy the needs of SSL_read() to continue the operation. In this case a call to SSL_get_error(3) with the return value of SSL_read() will yield SSL_ERROR_WANT_READ or SSL_ERROR_WANT_WRITE. As at any time a re-negotiation is possible, a call to SSL_read() can also cause write operations! The calling process then must repeat the call after taking appropriate action to satisfy the needs of SSL_read(). The action depends on the underlying BIO. When using a non-blocking socket, nothing is to be done, but select() can be used to check for the required condition. When using a buffering BIO, like a BIO pair, data must be written into or retrieved out of the BIO before being able to continue.
>
> SSL_pending(3) can be used to find out whether there are buffered bytes available for immediate retrieval. In this case SSL_read() can be called without blocking or actually receiving new data from the underlying socket.

总结下来说，`SSL_read`是根据record来处理数据的，如果一个record太大，就可能需要读取多个TCP包，在网络拥塞的情况下，如果某个TCP包丢失了，就会引起整个ssl record阻塞；那record是怎么产生的呢？  

参见[overlocking ssl](https://www.imperialviolet.org/2010/06/25/overclocking-ssl.html)  

> Packets and records
> SSL/TLS packages the bytes of the application protocol (HTTP in our case) into records. Each record has a signature and a header. Records are packed into packets and each packet has headers. The overhead of a record is typically 25 to 40 bytes (based on common ciphersuites) and the overhead of a packet is around 52 bytes. So it's vitally important not to send lots of small packets with small records in them.
> 
> I don't want to be seen to be picking on Bank Of America, it's honestly just the first site that I tried, but looking at their packets in Wireshark, we see many small records, often sent in a single packet. A quick sample of the record sizes: 638 bytes, 1363, 15628, 69, 182, 34, 18, … ==This is often caused because OpenSSL will build a record from each call to SSL_write and the kernel, with Nagle disabled, will send out packets to minimise latency==.
>
> This can be fixed with a couple of tricks: buffer in front of OpenSSL and don't make SSL_write calls with small amounts of data if you have more coming. Also, if code limitations mean that you are building small records in some cases, then use TCP_CORK to pack several of them into a packet.
>
> But don't make the records too large either! See the 15KB record that https://www.bankofamerica.com sent? None of that data can be parsed by the browser until the whole record has been received. As the congestion window opens up, those large records tend to span several windows and so there's an extra round trip of delay before the browser gets any of that data. Since the browser is pre-parsing the HTML for subresources, it'll delay discovery and cause more knock-on effects.
>
> So how large should records be? There's always going to be some uncertainty in that number because the size of the TCP header depends on the OS and the number of SACK blocks that need to be sent. In the ideal case, each packet is full and contains exactly one record. Start with a value of 1415 bytes for a non-padded ciphersuite (like RC4), or 1403 bytes for an AES based ciphersuite and look at the packets which result from that.  

其实就是在调用`SSL_write`时决定的，在写一个buffer时（non blocking情况下），第一次传入`SSL_write`的size参数就是record的大小（当然这个大小不能超过openssl的限制，即16K）,后续的`SSL_write`必须把前面一个record写完才会开始下一个record的写

以下几篇优化相关的文章：

1. [Ilya Grigorik - Optimizing TLS Record Size & Buffering Latency](https://www.igvita.com/2013/10/24/optimizing-tls-record-size-and-buffering-latency/)  
2. [cloudflare - Optimizing TLS over TCP to reduce latency](https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency/)  
3. [nginx patch from cloudflare](https://github.com/cloudflare/sslconfig/blob/master/patches/nginx__dynamic_tls_records.patch)  

几本书

1. [HTTPS权威指南](https://book.douban.com/subject/26869219/)  
2. [High Performance Browser Networking](https://hpbn.co/)  
3. [golang implementation](https://go-review.googlesource.com/c/go/+/19591)  
