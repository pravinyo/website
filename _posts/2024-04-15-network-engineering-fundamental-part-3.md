---
title: "Fundamentals of Network Engineering: The Cornerstone Protocol - TCP Detailed - Part 3"
author: pravin_tripathi
date: 2024-04-15 00:00:00 +0530
readtime: true
media_subpath: /assets/img/network-engineering-fundamental/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, network-engineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

## TCP (Transmission Control Protocol)

While UDP offers speed and simplicity, TCP provides reliability. This article delves into the details of the Transmission Control Protocol, its connections, structure, states, and trade-offs.

**Basic Characteristics:**
- Layer 4 protocol
- Process addressing via ports
- Controlled transmission (unlike UDP's "firehose" approach)
- Connection-oriented
- Requires handshake process
- 20-60 byte headers
- Stateful operation

**Common TCP Applications:**
- Reliable communications
- Remote shell connections
- Database connections
- Web communications
- Any bidirectional communication requiring reliability

**TCP Connection Details:**
- Operates at Layer 5 (Session)
- Represents an agreement between client and server
- Required before data transmission
- Identified by four properties:
  - Source IP
  - Source Port
  - Destination IP
  - Destination Port
- Sometimes called a socket or file descriptor
- Established via three-way handshake
- Features sequenced and ordered segments
- Includes segment acknowledgment
- Retransmits lost segments

![](image27.png)

![](image7.png)

![](image2.png)

![](image16.png)

![](image46.png)

![](image6.png)

**TCP Segment Structure:**
![](image45.png)
- Header is 20-60 bytes
- Encapsulated within IP packets as "data"
- Ports use 16 bits (range: 0-65535)
- Includes sequence numbers, acknowledgments, flow control mechanisms

![](image30.png)

**TCP Connection States:**
TCP maintains connection state on both client and server sides:
- Tracks window sizes, sequence numbers, and connection status
- Follows a defined state machine for connection establishment and termination
![](image26.png)

In the above diagram, we can see that 2 entity is interacting to close the TCP connection. Lets assume left side is our backend server and right side is client.

So. server sent `FIN_WAIT_1` to say I want to close the connection, client acknowledge that and moved to `CLOSE_WAIT` state.

In the meantime, client sent `FIN` request and moved to `LAST_ACK` state. Server receives it and acknowledge it by moving to `TIME WAIT` state. It waits for confirmation that acknowledge is received. If acknowledgement is not received as per spec, it will wait for 2 mins before sending another but it will not close the connection without confirmation.

As you see, server is maintaining the state information in file descriptor of the connection and that pipeline is open so port cannot be reused until closed. This takes finite amount of memory.

This is bad can slow down our server. One fix is to notify client to initiate this closure request so our server can close it and it will not have to go in time wait state. We don’t care about client most important thing is sever.

**TCP Advantages:**
- Guaranteed delivery
- Connection-oriented security
- Flow and congestion control
- Ordered packet delivery
- Resistance to spoofing

**TCP Disadvantages:**
- Larger header overhead than UDP
- Higher bandwidth requirements
- Stateful nature consumes memory
- Higher latency for certain workloads
- Complex behavior at low levels
- Head-of-line blocking issues


**Demo Example: Capture TCP segment using TCP DUMP**
```sh
> sudo tcpdump -n -v -i en0 src 93.184.215.14 or dst 93.184.215.14 and port 80

# Open example.com in browser

23:17:38.749801 IP (tos 0x0, ttl 64, id 0, offset 0, **flags [DF]**, proto TCP (6), length 64)

192.168.0.229.50445 > 93.184.215.14.80: Flags [SEW], cksum 0x2293 (correct), seq 1343525314, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 651251535 ecr 0,sackOK,eol], length 0

23:17:38.750104 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 64)

192.168.0.229.50446 > 93.184.215.14.80: Flags [SEW], cksum 0xbc5b (correct), seq 162874138, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 3938551452 ecr 0,sackOK,eol], length 0

23:17:38.961654 IP (tos 0x0, ttl 56, id 0, offset 0, flags [DF], proto TCP (6), length 60)

93.184.215.14.80 > 192.168.0.229.50446: Flags [S.E], cksum 0x586f (correct), seq 420561667, ack 162874139, win 28960, options [mss 1460,sackOK,TS val 1829649178 ecr 3938551452,nop,wscale 9], length 0

23:17:38.961746 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0xefbf (correct), ack 1, win 2058, options [nop,nop,TS val 3938551664 ecr 1829649178], length 0

23:17:38.961955 IP (tos 0x2,ECT(0), ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 496)

192.168.0.229.50446 > 93.184.215.14.80: Flags [P.], cksum 0xe434 (correct), seq 1:445, ack 1, win 2058, options [nop,nop,TS val 3938551664 ecr 1829649178], length 444: HTTP, length: 444

GET / HTTP/1.1
Host: example.com
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8

23:17:38.969121 IP (tos 0x0, ttl 56, id 0, offset 0, flags [DF], proto TCP (6), length 60)

93.184.215.14.80 > 192.168.0.229.50445: Flags [S.E], cksum 0x7b31 (correct), seq 3432239860, ack 1343525315, win 28960, options [mss 1460,sackOK,TS val 1829649179 ecr 651251535,nop,wscale 9], length 0

23:17:38.969207 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [.], cksum 0x127a (correct), ack 1, win 2058, options [nop,nop,TS val 651251755 ecr 1829649179], length 0

23:17:38.974101 IP (tos 0x0, ttl 56, id 30744, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0xf5d0 (correct), ack 445, win 59, options [nop,nop,TS val 1829649180 ecr 3938551664], length 0

23:17:38.974102 IP (tos 0x0, ttl 56, id 30745, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0xf5d0 (correct), ack 445, win 59, options [nop,nop,TS val 1829649180 ecr 3938551664], length 0

23:17:39.175389 IP (tos 0x2,ECT(0), ttl 56, id 30746, offset 0, flags [DF], proto TCP (6), length 1059)

93.184.215.14.80 > 192.168.0.229.50446: Flags [P.], cksum 0x02d3 (correct), seq 1:1008, ack 445, win 59, options [nop,nop,TS val 1829649199 ecr 3938551664], length 1007: HTTP, length: 1007

HTTP/1.1 200 OK
Content-Encoding: gzip
Age: 169439
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Sat, 28 Sep 2024 17:47:39 GMT
Etag: "3147526947+gzip"
Expires: Sat, 05 Oct 2024 17:47:39 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECAcc (nyd/D11B)
Vary: Accept-Encoding
X-Cache: HIT
Content-Length: 648

23:17:39.175500 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0xe938 (correct), ack 1008, win 2043, options [nop,nop,TS val 3938551878 ecr 1829649199], length 0

23:17:39.411028 IP (tos 0x2,ECT(0), ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 436)

192.168.0.229.50446 > 93.184.215.14.80: Flags [P.], cksum 0x54e0 (correct), seq 445:829, ack 1008, win 2048, options [nop,nop,TS val 3938552113 ecr 1829649199], length 384: HTTP, length: 384

GET /favicon.ico HTTP/1.1
Host: example.com
Connection: keep-alive
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Referer: http://example.com/
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8

23:17:39.421576 IP (tos 0x0, ttl 56, id 30747, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0xee72 (correct), ack 829, win 61, options [nop,nop,TS val 1829649224 ecr 3938552113], length 0

23:17:39.627672 IP (tos 0x2,ECT(0), ttl 56, id 30748, offset 0, flags [DF], proto TCP (6), length 1067)

93.184.215.14.80 > 192.168.0.229.50446: Flags [P.], cksum 0x128e (correct), seq 1008:2023, ack 829, win 61, options [nop,nop,TS val 1829649244 ecr 3938552113], length 1015: HTTP, length: 1015

HTTP/1.1 404 Not Found
Content-Encoding: gzip
Accept-Ranges: bytes
Age: 169189
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Sat, 28 Sep 2024 17:47:39 GMT
Expires: Sat, 05 Oct 2024 17:47:39 GMT
Last-Modified: Thu, 26 Sep 2024 18:47:50 GMT
Server: ECAcc (nyd/D157)
Vary: Accept-Encoding
X-Cache: 404-HIT
Content-Length: 648

23:17:39.627783 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0xe1db (correct), ack 2023, win 2032, options [nop,nop,TS val 3938552330 ecr 1829649244], length 0

23:17:59.013539 IP (tos 0x0, ttl 56, id 59881, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50445: Flags [.], cksum 0x1277 (correct), ack 1, win 57, options [nop,nop,TS val 1829651184 ecr 651251755], length 0

23:17:59.013593 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [.], cksum 0xc42d (correct), ack 1, win 2058, options [nop,nop,TS val 651271799 ecr 1829649179], length 0

23:17:59.632949 IP (tos 0x0, ttl 56, id 30749, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0xe1bd (correct), ack 829, win 61, options [nop,nop,TS val 1829651246 ecr 3938552330], length 0

23:17:59.633008 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0x93a6 (correct), ack 2023, win 2048, options [nop,nop,TS val 3938572335 ecr 1829649244], length 0

23:18:19.058009 IP (tos 0x0, ttl 56, id 59882, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50445: Flags [.], cksum 0xbc56 (correct), ack 1, win 57, options [nop,nop,TS val 1829653188 ecr 651271799], length 0

23:18:19.058123 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [.], cksum 0x75e2 (correct), ack 1, win 2058, options [nop,nop,TS val 651291842 ecr 1829649179], length 0

23:18:19.660677 IP (tos 0x0, ttl 56, id 30750, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0x8bc6 (correct), ack 829, win 61, options [nop,nop,TS val 1829653248 ecr 3938572335], length 0

23:18:19.660764 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0x456b (correct), ack 2023, win 2048, options [nop,nop,TS val 3938592362 ecr 1829649244], length 0

23:18:39.093461 IP (tos 0x0, ttl 56, id 59883, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50445: Flags [.], cksum 0x6637 (correct), ack 1, win 57, options [nop,nop,TS val 1829655192 ecr 651291842], length 0

23:18:39.093545 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [.], cksum 0x279e (correct), ack 1, win 2058, options [nop,nop,TS val 651311878 ecr 1829649179], length 0

23:18:39.613480 IP (tos 0x0, ttl 56, id 59884, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50445: Flags [F.], cksum 0x17be (correct), seq 1, ack 1, win 57, options [nop,nop,TS val 1829655243 ecr 651311878], length 0

23:18:39.613538 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [.], cksum 0x0de5 (correct), ack 2, win 2058, options [nop,nop,TS val 651312398 ecr 1829655243], length 0

23:18:39.693303 IP (tos 0x0, ttl 56, id 30751, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0x35b7 (correct), ack 829, win 61, options [nop,nop,TS val 1829655252 ecr 3938592362], length 0

23:18:39.693428 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0xf729 (correct), ack 2023, win 2048, options [nop,nop,TS val 3938612395 ecr 1829649244], length 0

23:18:45.029487 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50445 > 93.184.215.14.80: Flags [F.], cksum 0xf8bb (correct), seq 1, ack 2, win 2058, options [nop,nop,TS val 651317814 ecr 1829655243], length 0

23:18:45.044280 IP (tos 0x0, ttl 56, id 59885, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50445: Flags [.], cksum 0xfe6d (correct), ack 2, win 57, options [nop,nop,TS val 1829655786 ecr 651317814], length 0

23:18:59.734013 IP (tos 0x0, ttl 56, id 30752, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0xdfa1 (correct), ack 829, win 61, options [nop,nop,TS val 1829657256 ecr 3938612395], length 0

23:18:59.734099 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0xa8e1 (correct), ack 2023, win 2048, options [nop,nop,TS val 3938632435 ecr 1829649244], length 0

23:19:19.781744 IP (tos 0x0, ttl 56, id 30753, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.50446: Flags [.], cksum 0x8985 (correct), ack 829, win 61, options [nop,nop,TS val 1829659260 ecr 3938632435], length 0

23:19:19.781824 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.50446 > 93.184.215.14.80: Flags [.], cksum 0x5a91 (correct), ack 2023, win 2048, options [nop,nop,TS val 3938652483 ecr 1829649244], length 0
```

```sh
❯ sudo tcpdump -n -v -i en0 src 93.184.215.14 or dst 93.184.215.14 and port 80

tcpdump: listening on en0, link-type EN10MB (Ethernet), snapshot length 524288 bytes

00:19:55.933826 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 64)

192.168.0.229.51130 > 93.184.215.14.80: Flags [S], cksum 0x3da4 (correct), seq 2063463759, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 4247899513 ecr 0,sackOK,eol], length 0

00:19:55.933970 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 64)

192.168.0.229.51131 > 93.184.215.14.80: Flags [S], cksum 0xdd0c (correct), seq 3233684168, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 1273984025 ecr 0,sackOK,eol], length 0

00:19:56.136076 IP (tos 0x0, ttl 56, id 0, offset 0, flags [DF], proto TCP (6), length 60)

93.184.215.14.80 > 192.168.0.229.51130: Flags [S.], cksum 0xfbf6 (correct), seq 513304298, ack 2063463760, win 28960, options [mss 1460,sackOK,TS val 1830022887 ecr 4247899513,nop,wscale 9], length 0

00:19:56.136205 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x9310 (correct), ack 1, win 2058, options [nop,nop,TS val 4247899716 ecr 1830022887], length 0

00:19:56.136491 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 606)

192.168.0.229.51130 > 93.184.215.14.80: Flags [P.], cksum 0xd5a1 (correct), seq 1:555, ack 1, win 2058, options [nop,nop,TS val 4247899716 ecr 1830022887], length 554: HTTP, length: 554

GET / HTTP/1.1
Host: example.com
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8
If-None-Match: "3147526947+gzip"
If-Modified-Since: Thu, 17 Oct 2019 07:18:26 GMT
00:19:56.142725 IP (tos 0x0, ttl 56, id 0, offset 0, flags [DF], proto TCP (6), length 60)

93.184.215.14.80 > 192.168.0.229.51131: Flags [S.], cksum 0xc751 (correct), seq 4259666857, ack 3233684169, win 28960, options [mss 1460,sackOK,TS val 1830022888 ecr 1273984025,nop,wscale 9], length 0

00:19:56.142809 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0x5e65 (correct), ack 1, win 2058, options [nop,nop,TS val 1273984234 ecr 1830022888], length 0

00:19:56.151378 IP (tos 0x0, ttl 56, id 34818, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x98b3 (correct), ack 555, win 59, options [nop,nop,TS val 1830022889 ecr 4247899716], length 0

00:19:56.151612 IP (tos 0x0, ttl 56, id 34819, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x98b3 (correct), ack 555, win 59, options [nop,nop,TS val 1830022889 ecr 4247899716], length 0

00:19:56.347316 IP (tos 0x0, ttl 56, id 34820, offset 0, flags [DF], proto TCP (6), length 336)

93.184.215.14.80 > 192.168.0.229.51130: Flags [P.], cksum 0xc646 (correct), seq 1:285, ack 555, win 59, options [nop,nop,TS val 1830022908 ecr 4247899716], length 284: HTTP, length: 284

HTTP/1.1 304 Not Modified
Age: 173184
Cache-Control: max-age=604800
Date: Sat, 28 Sep 2024 18:49:56 GMT
Etag: "3147526947+gzip"
Expires: Sat, 05 Oct 2024 18:49:56 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECAcc (nyd/D12F)
Vary: Accept-Encoding
X-Cache: HIT

00:19:56.347397 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x8ee6 (correct), ack 285, win 2054, options [nop,nop,TS val 4247899927 ecr 1830022908], length 0

00:20:16.222862 IP (tos 0x0, ttl 56, id 46403, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [.], cksum 0x5e5f (correct), ack 1, win 57, options [nop,nop,TS val 1830024896 ecr 1273984234], length 0

00:20:16.222933 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0x0ff5 (correct), ack 1, win 2058, options [nop,nop,TS val 1274004314 ecr 1830022888], length 0

00:20:16.351082 IP (tos 0x0, ttl 56, id 34821, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x8ee1 (correct), ack 555, win 59, options [nop,nop,TS val 1830024909 ecr 4247899927], length 0

00:20:16.351160 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x40c3 (correct), ack 285, win 2054, options [nop,nop,TS val 4247919930 ecr 1830022908], length 0

00:20:36.262327 IP (tos 0x0, ttl 56, id 46404, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [.], cksum 0x081b (correct), ack 1, win 57, options [nop,nop,TS val 1830026900 ecr 1274004314], length 0

00:20:36.262441 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0xc1ad (correct), ack 1, win 2058, options [nop,nop,TS val 1274024353 ecr 1830022888], length 0

00:20:36.381400 IP (tos 0x0, ttl 56, id 34822, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x38eb (correct), ack 555, win 59, options [nop,nop,TS val 1830026912 ecr 4247919930], length 0

00:20:36.381507 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0xf284 (correct), ack 285, win 2054, options [nop,nop,TS val 4247939960 ecr 1830022908], length 0

00:20:56.418635 IP (tos 0x0, ttl 56, id 46405, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [.], cksum 0xb1ff (correct), ack 1, win 57, options [nop,nop,TS val 1830028904 ecr 1274024353], length 0

00:20:56.418750 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0x72f1 (correct), ack 1, win 2058, options [nop,nop,TS val 1274044509 ecr 1830022888], length 0

00:20:56.421745 IP (tos 0x0, ttl 56, id 34823, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0xe2d8 (correct), ack 555, win 59, options [nop,nop,TS val 1830028916 ecr 4247939960], length 0

00:20:56.421851 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0xa43c (correct), ack 285, win 2054, options [nop,nop,TS val 4247960000 ecr 1830022908], length 0

00:20:56.670459 IP (tos 0x0, ttl 56, id 46406, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [F.], cksum 0x631d (correct), seq 1, ack 1, win 57, options [nop,nop,TS val 1830028940 ecr 1274044509], length 0

00:20:56.670570 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0x5a50 (correct), ack 2, win 2058, options [nop,nop,TS val 1274044761 ecr 1830028940], length 0

00:21:16.462443 IP (tos 0x0, ttl 56, id 34824, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x8cbc (correct), ack 555, win 59, options [nop,nop,TS val 1830030920 ecr 4247960000], length 0

00:21:16.462556 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x55f4 (correct), ack 285, win 2054, options [nop,nop,TS val 4247980040 ecr 1830022908], length 0

00:21:16.672657 IP (tos 0x0, ttl 56, id 46407, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [.], cksum 0x5a51 (correct), ack 1, win 57, options [nop,nop,TS val 1830030941 ecr 1274044761], length 0

00:21:16.672762 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [.], cksum 0x0c2f (correct), ack 2, win 2058, options [nop,nop,TS val 1274064762 ecr 1830028940], length 0

00:21:34.146250 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51131 > 93.184.215.14.80: Flags [F.], cksum 0xc7eb (correct), seq 1, ack 2, win 2058, options [nop,nop,TS val 1274082236 ecr 1830028940], length 0

00:21:34.160178 IP (tos 0x0, ttl 56, id 46408, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51131: Flags [.], cksum 0xc117 (correct), ack 2, win 57, options [nop,nop,TS val 1830032689 ecr 1274082236], length 0

00:21:36.502414 IP (tos 0x0, ttl 56, id 34825, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x36a0 (correct), ack 555, win 59, options [nop,nop,TS val 1830032924 ecr 4247980040], length 0

00:21:36.502530 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x07ac (correct), ack 285, win 2054, options [nop,nop,TS val 4248000080 ecr 1830022908], length 0

00:21:56.583501 IP (tos 0x0, ttl 56, id 34826, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0xe083 (correct), ack 555, win 59, options [nop,nop,TS val 1830034928 ecr 4248000080], length 0

00:21:56.583534 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0xb93b (correct), ack 285, win 2054, options [nop,nop,TS val 4248020160 ecr 1830022908], length 0

00:22:16.622925 IP (tos 0x0, ttl 56, id 34827, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x8a3e (correct), ack 555, win 59, options [nop,nop,TS val 1830036933 ecr 4248020160], length 0

00:22:16.623041 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x6af4 (correct), ack 285, win 2054, options [nop,nop,TS val 4248040199 ecr 1830022908], length 0

00:22:36.633508 IP (tos 0x0, ttl 56, id 34828, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x3423 (correct), ack 555, win 59, options [nop,nop,TS val 1830038937 ecr 4248040199], length 0

00:22:36.633621 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x1cca (correct), ack 285, win 2054, options [nop,nop,TS val 4248060209 ecr 1830022908], length 0

00:22:56.711840 IP (tos 0x0, ttl 56, id 34829, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0xde21 (correct), ack 555, win 59, options [nop,nop,TS val 1830040944 ecr 4248060209], length 0

00:22:56.711902 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0xce5b (correct), ack 285, win 2054, options [nop,nop,TS val 4248080287 ecr 1830022908], length 0

00:22:57.285262 IP (tos 0x0, ttl 56, id 34830, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [F.], cksum 0x8f7e (correct), seq 285, ack 555, win 59, options [nop,nop,TS val 1830040995 ecr 4248080287], length 0

00:22:57.285330 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [.], cksum 0x8575 (correct), ack 286, win 2054, options [nop,nop,TS val 4248080861 ecr 1830040995], length 0

00:23:04.163282 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.51130 > 93.184.215.14.80: Flags [F.], cksum 0x6a97 (correct), seq 555, ack 286, win 2054, options [nop,nop,TS val 4248087738 ecr 1830040995], length 0

00:23:04.204860 IP (tos 0x0, ttl 56, id 34831, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.51130: Flags [.], cksum 0x6fa7 (correct), ack 556, win 59, options [nop,nop,TS val 1830041694 ecr 4248087738], length 0
```

```sh
tcpdump: listening on en0, link-type EN10MB (Ethernet), snapshot length 524288 bytes

12:01:12.375168 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 424)

192.168.0.229.49654 > 93.184.215.14.80: Flags [P.], cksum 0xaf39 (correct), seq 1764312273:1764312645, ack 2387558194, win 2048, options [nop,nop,TS val 3256430779 ecr 1834226902], length 372: HTTP, length: 372

GET / HTTP/1.1
Host: example.com
Upgrade-Insecure-Requests: 1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.6 Safari/605.1.15
Accept-Language: en-IN,en-GB;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate
Connection: keep-alive

12:01:12.399203 IP (tos 0x0, ttl 56, id 12186, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49654: Flags [.], cksum 0xbdd8 (correct), ack 372, win 61, options [nop,nop,TS val 1834230515 ecr 3256430779], length 0

12:01:12.579540 IP (tos 0x0, ttl 56, id 12187, offset 0, flags [DF], proto TCP (6), length 1059)

93.184.215.14.80 > 192.168.0.229.49654: Flags [P.], cksum 0xcee2 (correct), seq 1:1008, ack 372, win 61, options [nop,nop,TS val 1834230533 ecr 3256430779], length 1007: HTTP, length: 1007

HTTP/1.1 200 OK
Content-Encoding: gzip
Age: 384315
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Sun, 29 Sep 2024 06:31:12 GMT
Etag: "3147526947+gzip"
Expires: Sun, 06 Oct 2024 06:31:12 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECAcc (nyd/D15A)
Vary: Accept-Encoding
X-Cache: HIT
Content-Length: 648

12:01:12.579618 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49654 > 93.184.215.14.80: Flags [.], cksum 0xb157 (correct), ack 1008, win 2032, options [nop,nop,TS val 3256430984 ecr 1834230533], length 0

12:01:16.233203 IP (tos 0x0, ttl 56, id 13727, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49655: Flags [.], cksum 0xb37c (correct), ack 3442544668, win 59, options [nop,nop,TS val 1834230888 ecr 508911710], length 0

12:01:16.233269 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49655 > 93.184.215.14.80: Flags [.], cksum 0x6ca0 (correct), ack 1, win 2048, options [nop,nop,TS val 508931867 ecr 1834226881], length 0

12:01:32.617515 IP (tos 0x0, ttl 56, id 12188, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49654: Flags [.], cksum 0xb138 (correct), ack 372, win 61, options [nop,nop,TS val 1834232536 ecr 3256430984], length 0

12:01:32.617631 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49654 > 93.184.215.14.80: Flags [.], cksum 0x6301 (correct), ack 1008, win 2048, options [nop,nop,TS val 3256451022 ecr 1834230533], length 0

12:01:35.635764 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49655 > 93.184.215.14.80: Flags [F.], cksum 0x20d4 (correct), seq 1, ack 1, win 2048, options [nop,nop,TS val 508951270 ecr 1834226881], length 0

12:01:35.695220 IP (tos 0x0, ttl 56, id 13728, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49655: Flags [.], cksum 0x114d (correct), ack 2, win 59, options [nop,nop,TS val 1834232845 ecr 508951270], length 0

12:01:35.843950 IP (tos 0x0, ttl 56, id 13729, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49655: Flags [F.], cksum 0x113e (correct), seq 1, ack 2, win 59, options [nop,nop,TS val 1834232859 ecr 508951270], length 0

12:01:35.844081 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49655 > 93.184.215.14.80: Flags [.], cksum 0x08a9 (correct), ack 2, win 2048, options [nop,nop,TS val 508951478 ecr 1834232859], length 0

12:01:52.688349 IP (tos 0x0, ttl 56, id 12189, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49654: Flags [.], cksum 0x5b1e (correct), ack 372, win 61, options [nop,nop,TS val 1834234540 ecr 3256451022], length 0

12:01:52.688463 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49654 > 93.184.215.14.80: Flags [.], cksum 0x149a (correct), ack 1008, win 2048, options [nop,nop,TS val 3256471093 ecr 1834230533], length 0

12:02:05.636124 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49654 > 93.184.215.14.80: Flags [F.], cksum 0xe205 (correct), seq 372, ack 1008, win 2048, options [nop,nop,TS val 3256484040 ecr 1834230533], length 0

12:02:05.694335 IP (tos 0x0, ttl 56, id 12190, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49654: Flags [.], cksum 0xd508 (correct), ack 373, win 61, options [nop,nop,TS val 1834235845 ecr 3256484040], length 0

12:02:05.836201 IP (tos 0x0, ttl 56, id 12191, offset 0, flags [DF], proto TCP (6), length 52)

93.184.215.14.80 > 192.168.0.229.49654: Flags [F.], cksum 0xd4f9 (correct), seq 1008, ack 373, win 61, options [nop,nop,TS val 1834235859 ecr 3256484040], length 0

12:02:05.836396 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)

192.168.0.229.49654 > 93.184.215.14.80: Flags [.], cksum 0xcc6d (correct), ack 1009, win 2048, options [nop,nop,TS val 3256484241 ecr 1834235859], length 0
```

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*