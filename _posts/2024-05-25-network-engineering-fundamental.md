---
title: "Fundamentals of Network Engineering"
author: pravin_tripathi
date: 2024-05-25 00:00:00 +0530
readtime: true
media_subpath: /assets/img/network-engineering-fundamental/
file_document_path: "/assets/document/attachment/network-engineering-fundamental/2210.00714v2.pdf"
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

In today's interconnected world, understanding network engineering fundamentals is essential for any tech professional. This guide breaks down complex networking concepts into digestible components, from client-server architecture to TCP/IP protocols.

## Client-Server Architecture: The Foundation of Modern Computing

Client-server architecture divides computing workloads between powerful servers and lighter-weight clients, creating a more efficient system:

1. **Problem it solves**: Machines are expensive, and applications are complex
2. **Solution**: Separate applications into two components
3. **Workload distribution**: Expensive processing happens on the server
4. **Interaction model**: Clients call servers to perform resource-intensive tasks
5. **Result**: Remote Procedure Call (RPC) was born

**Key Benefits:**
- Servers can utilize powerful hardware for intensive operations
- Clients operate effectively on commodity hardware
- Clients can still handle lightweight tasks independently
- Clients don't require all dependencies locally
- **However**: We need a standardized communication model

This is where networking models come into play.

## The OSI Model: Why We Need Communication Standards

The Open Systems Interconnection (OSI) model provides a conceptual framework that standardizes network communications. But why do we need such a model?

### Advantages of a Standardized Communication Model:

**1. Application Agnosticism**
- Without standards, applications would require knowledge of every underlying network medium
- Imagine developing different versions of your app for WiFi, Ethernet, LTE, and fiber

**2. Simplified Network Equipment Management**
- Standards make upgrading network equipment more straightforward
- Interoperability between different vendors and technologies

**3. Decoupled Innovation**
- Each layer can evolve independently
- Improvements in one layer don't require changes in others

## The OSI Model: 7 Layers Explained

The OSI model divides networking into seven distinct layers, each handling specific functions:

### Layer 7 - Application
- **Function**: Interfaces directly with applications and users
- **Examples**: HTTP, FTP, gRPC, SMTP
- **Role**: Provides network services to applications

### Layer 6 - Presentation
- **Function**: Data translation and encryption
- **Examples**: Encoding, serialization, encryption/decryption
- **Role**: Ensures data is in a usable format for the application layer

### Layer 5 - Session
- **Function**: Establishes, manages, and terminates connections
- **Examples**: Connection establishment, TLS
- **Role**: Maintains dialogue between devices

### Layer 4 - Transport
- **Function**: End-to-end communication and data flow control
- **Examples**: TCP, UDP
- **Role**: Ensures complete data transfer

### Layer 3 - Network
- **Function**: Logical addressing and routing
- **Examples**: IP (IPv4, IPv6)
- **Role**: Determines how data is sent to the receiving device

### Layer 2 - Data Link
- **Function**: Physical addressing and media access control
- **Examples**: Ethernet frames, MAC addresses
- **Role**: Transfers data between network entities

### Layer 1 - Physical
- **Function**: Transmission of raw bit stream
- **Examples**: Electric signals, fiber optics, radio waves
- **Role**: Sends and receives data through the physical medium

## Data Flow Through the OSI Model

### Sending Data: A POST Request to an HTTPS Webpage

#### Layer 7 - Application
- POST request with JSON data is created for an HTTPS server

#### Layer 6 - Presentation
- JSON data is serialized into flat byte strings

#### Layer 5 - Session
- Request to establish TCP connection and TLS session

#### Layer 4 - Transport
- Sends SYN request targeting port 443

#### Layer 3 - Network
- SYN is placed in IP packet(s) with source/destination IP addresses

#### Layer 2 - Data Link
- Each packet is encapsulated in a frame with source/destination MAC addresses

#### Layer 1 - Physical
- Frames are converted into signals appropriate for the physical medium:
  - Radio signals for WiFi
  - Electric signals for Ethernet
  - Light pulses for fiber optic connections

### Receiving Data: The Reverse Journey

#### Layer 1 - Physical
- Physical signals (radio, electric, light) are received and converted to digital bits

#### Layer 2 - Data Link
- Bits are assembled into frames

#### Layer 3 - Network
- Frames are assembled into IP packets

#### Layer 4 - Transport
- IP packets are assembled into TCP segments
- Handles congestion control, flow control, and retransmission for TCP
- For SYN packets, processing may stop here as connection establishment is still in progress

#### Layer 5 - Session
- Connection session is identified or established
- Only reached after the three-way handshake is complete

#### Layer 6 - Presentation
- Byte strings are deserialized back to JSON for application consumption

#### Layer 7 - Application
- Application processes the JSON POST request
- Triggers appropriate handlers (like Express.js or Apache request events)

> Note: The clean separation between layers isn't always clear-cut in real-world implementations.

## The TCP/IP Model: A Practical Alternative

![](image42.png)

![](image22.png)

![](image40.png)

![](image1.png)

The TCP/IP model simplifies the OSI model into four practical layers:

1. **Application Layer** (Combines OSI Layers 5, 6, and 7)
   - Application, presentation, and session functionality

2. **Transport Layer** (OSI Layer 4)
   - End-to-end communication (TCP, UDP)

3. **Internet Layer** (OSI Layer 3)
   - Routing and logical addressing (IP)

4. **Network Interface Layer** (OSI Layer 2)
   - Physical addressing and media access

Note that the physical layer isn't officially included in the TCP/IP model.

## Network Protocols: The Language of the Internet

### ICMP (Internet Control Message Protocol)

- **Purpose**: Designed for network diagnostic and control messages
- **Features**:
  - Transmits informational messages like "host unreachable," "port unreachable," or "fragmentation needed"
  - Reports when packets expire (caught in routing loops)
  - Uses IP directly (not TCP or UDP)
  - Underpins tools like ping and traceroute
  - Requires no open ports or listeners
  - Helps with MTU (Maximum Transmission Unit) discovery
  - If blocked, creates a "TCP black hole" where critical information cannot reach source machines

**Example: Ping pravin.dev**

Different `tcpdump` command to check ip packets
```sh
# ping any server for which you want to see packets.
ping pravin.dev

# use this to see packet dump for en0 interface
tcpdump -i en0 arp
tcpdump -n -i en0 icmp
tcpdump -n -v -i en0 icmp

# filter packets using source and destination IP address
tcpdump -n -v -i en0 icmp src 185.199.111.153 or dst 185.199.111.153
tcpdump -n -v -i en0 src 185.199.111.153 or dst 185.199.111.153
```

Routing of IP packets through various devices like routers, switches, etc.

![](image12.png)

### UDP (User Datagram Protocol)

**Basic Characteristics:**
- Layer 4 protocol
- Addresses processes using ports
- Simple protocol for data transmission
- Requires no prior communication (stateless)
- Compact 8-byte header (vs. IP's 20-byte header)

**Common UDP Applications:**
- Video streaming
- VPN services
- DNS queries
- WebRTC communications

**UDP Advantages:**
- Simplicity
- Small header size and bandwidth efficiency
- Stateless operation
- Lower memory consumption
- Low latency with no handshakes or delivery guarantees

**UDP Disadvantages:**
- No acknowledgment mechanism
- No guaranteed delivery
- Connection-less (anyone can send data)
- No flow or congestion control
- No packet ordering
- Potential security vulnerabilities (easily spoofed)

**Demo Example:**

**Terminal 1:**
```sh
❯ tcpdump -n -v -i en0 src 8.8.8.8 or dst 8.8.8.8 #(Listen to Google DNS )
```

**Terminal 2:**
```sh
❯ nslookup pravin.dev 8.8.8.8

Server: 8.8.8.8
Address: 8.8.8.8#53
Non-authoritative answer:

Name: pravin.dev
Address: 185.199.111.153

Name: pravin.dev
Address: 185.199.109.153

Name: pravin.dev
Address: 185.199.108.153

Name: pravin.dev
Address: 185.199.110.153
```

**Back to Terminal 1:**
```sh
tcpdump: listening on en0, link-type EN10MB (Ethernet), snapshot length 524288 bytes

22:13:22.540384 IP (tos 0x0, ttl 64, id 51364, offset 0, flags [none], proto UDP (17), length 56)

192.168.0.229.61141 > 8.8.8.8.53: 37528+ A? pravin.dev. (28)

22:13:22.573255 IP (tos 0x0, ttl 55, id 35405, offset 0, flags [none], proto UDP (17), length 120)

8.8.8.8.53 > 192.168.0.229.61141: 37528 4/0/0 pravin.dev. A 185.199.111.153, pravin.dev. A 185.199.109.153, pravin.dev. A 185.199.108.153, pravin.dev. A 185.199.110.153 (92)
```

### TCP (Transmission Control Protocol)

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

## Network Address Translation (NAT)

NAT allows multiple devices on a local network to share a single public IP address:

- Maps private IP addresses to public IP addresses
- Essential for IPv4 address conservation
- Creates challenges for certain applications and protocols
- Has implications for direct connectivity between devices

![](image10.png)

![](image18.png)

## Real-World Implications of TCP Connection States

When a TCP connection closes, both client and server go through several states:

1. The server sends `FIN_WAIT_1` to initiate connection closure
2. The client acknowledges and enters `CLOSE_WAIT` state
3. The client sends its own `FIN` request and enters `LAST_ACK` state
4. The server acknowledges and enters `TIME_WAIT` state, holding resources until confirmation

This state maintenance:
- Consumes memory in server file descriptors
- Keeps connections open even after logical closure
- Can impact server performance during high-volume connection handling

An optimization involves having clients initiate closure requests, preventing servers from entering the `TIME_WAIT` state.

## TCP Advantages:
- Guarantee delivery
- No one can send data without prior knowledge
- Flow Control and Congestion Control
- Ordered Packets no corruption or app level work
- Secure and can’t be easily spoofed

## TCP Disadvantages:
- Large header overhead compared to UDP
- More bandwidth
- Stateful - consumes memory on server and client
- Considered high latency for certain workloads (Slow start/ congestion/ acks)
- Does too much at a low level (hence QUIC)
- Single connection to send multiple streams of data (HTTP requests)
- Stream 1 has nothing to do with Stream 2
- Both Stream 1 and Stream 2 packets must arrive
- TCP Meltdown
- Not a good candidate for VPN

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

## Transport Layer Security (TLS): Securing Network Communication

Modern internet security relies heavily on encryption, particularly through Transport Layer Security (TLS). Let's explore how we've evolved from basic HTTP to secure communications:

### The Evolution of Web Security
1. **Vanilla HTTP**: The original web protocol - unencrypted and vulnerable
2. **HTTPS**: HTTP with encryption through SSL/TLS
3. **TLS 1.2 Handshake**: Two round trips required to establish secure connection
4. **Diffie-Hellman Key Exchange**: A secure method for sharing encryption keys
5. **TLS 1.3 Improvements**: Reduced to one round trip (or even zero in some cases)

![](image35.png)
![](image28.png)

### Why We Need TLS

TLS serves several critical purposes:

- **Encryption**: We use symmetric key algorithms for encrypting data
- **Key Exchange**: We need a secure way to exchange symmetric keys
- **Server Authentication**: Verifying the server's identity is crucial
- **Extensions**: Features like Server Name Indication (SNI), pre-shared keys, and 0-RTT (zero round trip time)

![](image43.png)
![](image39.png)
![](image14.png)
![](image36.png)

TLS establishes a secure channel through a carefully choreographed handshake process:

1. Client and server agree on the cryptographic algorithms to use
2. Authentication occurs through certificates
3. Session keys are established securely
4. Encrypted communication can begin

The key advantages of TLS 1.3 over previous versions include:
- Faster connections (fewer round trips)
- Improved security with deprecated legacy algorithms
- Forward secrecy by default
- Simplified negotiation process

## MSS/MTU and Path MTU: Understanding Packet Size Limitations

One of the fundamental challenges in networking is determining how large data packets can be:

#### The Layering of Network Units
- TCP (Layer 4): Data is organized into **segments**
- IP (Layer 3): Segments are encapsulated into **packets** with headers
- Data Link (Layer 2): Packets are placed into **frames**

**The critical constraint**: The frame size is fixed based on network configuration, which ultimately determines segment size.

### Hardware MTU Explained

The **Maximum Transmission Unit (MTU)** defines the size of a frame:
- It's a property of the network interface (default is typically 1500 bytes)
- Some networks support "jumbo frames" up to 9000 bytes
- Larger frames can reduce latency by requiring fewer transmissions
- However, larger frames are more vulnerable to corruption on unstable networks

### IP Packets and MTU Relationship

- IP MTU typically equals the hardware MTU
- Ideally, one IP packet should fit within a single frame
- If necessary, IP fragmentation splits large packets into multiple frames
- Fragmentation can impact performance

### Maximum Segment Size (MSS)

MSS is calculated based on the MTU:

**MSS = MTU - IP Headers - TCP Headers**

Using standard values: **MSS = 1500 - 20 - 20 = 1460 bytes**

This means:
- A 1460-byte payload fits perfectly in one MSS
- Which fits in one IP packet
- Which fits in one frame
- Resulting in optimal transmission efficiency

![](image19.png)
From: [https://learningnetwork.cisco.com/s/question/0D53i00000Kt7CXCAZ/mtu-vs-pdu](https://learningnetwork.cisco.com/s/question/0D53i00000Kt7CXCAZ/mtu-vs-pdu)

![](image31.png)

### Summary of MTU and MSS Concepts:
- MTU is the maximum transmission unit at the device level
- MSS is the maximum segment size at layer 4
- Larger segments reduce latency and overhead
- Path MTU discovery finds the lowest MTU along a network path using ICMP
- Flow and congestion control mechanisms still allow multiple segments to be sent before acknowledgment

### Nagle's Algorithm: Efficiency vs. Latency

Nagle's algorithm was designed to improve network efficiency by reducing the number of small packets sent:

- **Origin**: Created during the telnet era when sending single bytes was common
- **Purpose**: Combines small segments into single, larger ones
- **Function**: Client waits until it has a full MSS before sending, or until all previous segments are acknowledged
- **Benefit**: Reduces wasted bandwidth from 40-byte headers (IP + TCP) carrying only a few bytes of actual data
- **Trade-off**: Can introduce delays, especially in TLS communications
- **Real-world impact**: Applications like Curl has disabled this feature as it started observing delay in TLS as Nagle’s algo expect packet to be completely filled before sending it until acknowledgement is received. This wait is viewed as delay or performance lag.

![](image3.png)
![](image8.png)
![](image20.png)

### Delayed Acknowledgement

Similar to Nagle's algorithm, delayed acknowledgment aims to reduce packet overhead:

- Receivers don't immediately acknowledge every segment
- Instead, they wait for either:
  - Another segment to arrive (allowing one ACK to cover two segments)
  - A timeout (typically 200-500ms)

![](image37.png)
![](image9.png)

While this reduces network traffic, it can cause performance issues when combined with Nagle's algorithm, creating a "mini-deadlock" situation where both sides are waiting for the other.

You can disable delayed acknowledgments using the TCP_QUICKACK option, making segments acknowledge faster at the cost of additional network traffic. Segments will be acknowledged “quicker”

## The Cost of Connections

### Why Connections Are Expensive

Establishing TCP connections incurs significant costs:

- **Handshake Overhead**: The three-way TCP handshake takes time
- **Distance Penalty**: Greater physical distance between endpoints increases latency
- **Slow Start**: TCP doesn't reach full throughput immediately
- **Control Mechanisms**: Congestion and flow control limit performance
- **Algorithm Impact**: Nagle's algorithm and delayed acknowledgments add latency
- **Termination Cost**: Connection teardown is also resource-intensive

### Connection Pooling: A Critical Optimization

To mitigate connection costs, most database backends and reverse proxies implement connection pooling:

- Pre-establish multiple TCP connections to backend services
- Keep connections open and ready for use
- Serve incoming requests using these "warm" connections
- Allow slow start to complete ahead of time
- Close connections only when absolutely necessary

This approach dramatically improves response times for clients.

### Eager vs. Lazy Loading

Two paradigms for resource management affect connection performance:

**Eager Loading**:
- Load everything and keep it ready
- Slower startup but faster request handling
- Some applications send "warm-up" data to trigger slow start
- Caution needed regarding bandwidth and scalability implications

**Lazy Loading**:
- Load resources only when needed
- Fast startup but initially slower requests
- Resources consumed only as necessary

### TCP Fast Open (TFO)

TCP Fast Open addresses the handshake latency problem:

- **Challenge**: Traditional TCP handshake is slow
- **Solution**: Allow data transmission during the handshake
- **Method**: Use a predetermined token for servers you've previously connected to
- **Result**: Reduced latency for repeat connections
- **Availability**: Enabled by default in Linux 3.13+
- **Usage**: Can be enabled in tools like curl with `--tcp-fastopen`
- **Limitation**: Still subject to TCP slow start

> You can take advantage of this feature to send early data

![](image34.png)

### TCP Head-of-Line (HOL) Blocking in HTTP/1 and HTTP/2

TCP's reliable delivery mechanism can create performance bottlenecks:

- Lost packets block all subsequent data until retransmission succeeds
- In HTTP/1.1, browsers open multiple TCP connections to work around this limitation
- HTTP/2 uses a single TCP connection with multiple streams
- While HTTP/2 avoids application-level HOL blocking, it's still vulnerable to TCP-level HOL blocking
- HTTP/3 addresses this by using QUIC, which is built on UDP

![](image24.png)
![](image29.png)

## Proxy Systems: Intermediaries in Network Communication

### Proxies: Client-Side Intermediaries

A proxy acts as an intermediary for client requests:

- Establishes new TCP connections with destination servers
- Sends modified requests on the client's behalf
- The destination server is aware of the proxy but not the original client
- May include headers like `X-Forwarded-For` to communicate client details

**Key Use Cases for Proxies:**
- Content caching
- Anonymous browsing
- Request/response logging
- Content filtering and site blocking
- Microservice architecture support

### Reverse Proxies: Server-Side Intermediaries

Unlike regular proxies, reverse proxies hide server details from clients:

- Clients connect to the reverse proxy, unaware of backend servers
- The reverse proxy forwards requests to appropriate backend servers
- All client communication occurs through the reverse proxy

**Important**: While all load balancers are reverse proxies, not all reverse proxies are load balancers.

Systems can employ both proxies and reverse proxies in complex arrangements.

**Key Use Cases for Reverse Proxies:**
- Content caching (CDNs)
- Load balancing
- API gateway/ingress control
- Canary deployments (directing percentage of traffic to new servers)
- Microservice architecture management

## Load Balancing: Layer 4 vs Layer 7

Load balancing is a critical component of modern web infrastructure, allowing traffic to be distributed across multiple servers for improved performance, reliability, and scalability. However, not all load balancers are created equal – their capabilities differ significantly based on which OSI layer they operate at.

![](image44.png)
![](image4.png)

### Layer 4 Load Balancing: Transport Layer Intelligence

Layer 4 load balancers operate at the transport layer, making routing decisions based on TCP/IP information:

- **Function**: Routes packets based on source/destination IP addresses and ports
- **Connection Handling**: Maintains state so requests from the same client go to the same server
- **Data Management**: Can perform buffering to optimize packet sizes between clients and servers

**Real-World Example**: 
1. When a client with MTU of 1460 connects through a load balancer to a server network that supports larger MTUs, the load balancer can buffer and consolidate packets to utilize the higher-capacity backend connection. 
2. The IP packet is fragmented and we don’t want the backend to receive fragmented packets as it can support larger packets which optimize the backend but yeah it might introduce some latency to the request.

![](image48.png)
It warms up.

![](image41.png)

Layer 4 is stateful so requests from the same client goes to the same server.

Load balancer acts like a router and navigates the packet to the appropriate destination. It’s like a sticky thing

![](image25.png)
![](image11.png)
![](image5.png)
![](image32.png)

When a new connection request comes, it will pick a different backend server as per the LB logic.

![](image15.png)

**How Layer 4 Load Balancers Work**:
1. Client initiates a TCP connection to the load balancer
2. Load balancer selects a backend server according to its algorithm
3. Load balancer establishes a connection to the chosen server
4. Using NAT (Network Address Translation), the load balancer forwards traffic between client and server
5. For subsequent connections, different backend servers may be selected

#### Layer 4 Load Balancer Advantages:
- **Simplicity**: Straightforward load balancing without complex logic
- **Efficiency**: No deep packet inspection means lower processing overhead
- **Security**: Doesn't read content payloads, reducing exposure
- **Protocol Agnostic**: Works with any TCP/UDP-based protocol
- **Connection Efficiency**: Maintains a single TCP connection (with NAT)

#### Layer 4 Load Balancer Disadvantages:
- **Limited Intelligence**: Cannot make routing decisions based on content
- **Microservice Challenges**: Difficult to route to specific microservices
- **Connection Stickiness**: Tied to the entire connection rather than individual requests
- **No Caching**: Cannot cache content
- **Protocol Unawareness**: May allow bypassing of certain security rules

### Layer 7 Load Balancing: Application Layer Intelligence

Layer 7 load balancers operate at the application layer, making sophisticated routing decisions based on the actual content of traffic:

- **Function**: Routes requests based on URLs, headers, cookies, and message content
- **Protocol Awareness**: Understands HTTP/HTTPS, WebSocket, and other application protocols
- **Connection Handling**: Terminates connections from clients and creates new ones to backend servers

![](image21.png)
![](image13.png)
![](image38.png)
![](image23.png)

**How Layer 7 Load Balancers Work**:
1. Client establishes a TCP connection to the load balancer
2. If HTTPS, a TLS handshake occurs, and the load balancer decrypts the traffic
3. Load balancer inspects the request content (URL, headers, etc.)
4. Based on routing rules, the load balancer selects an appropriate backend server
5. Load balancer establishes a connection to the chosen server
6. Request is forwarded, potentially with modifications
7. Response follows the reverse path

#### Layer 7 Load Balancer Advantages:
- **Intelligent Routing**: Can direct traffic based on content and application-specific attributes
- **Caching Capability**: Can cache responses for improved performance
- **Microservice Support**: Excellent for routing to specific microservices based on request paths
- **API Gateway Functionality**: Can implement API gateway logic
- **Authentication**: Can handle authentication before requests reach backend servers

#### Layer 7 Load Balancer Disadvantages:
- **Resource Intensity**: More expensive computationally due to content inspection
- **Encryption Handling**: Must terminate and decrypt TLS connections
- **Connection Management**: Creates two separate TCP connections per request
- **Certificate Requirements**: Needs access to TLS certificates
- **Buffering Needs**: Must buffer data for inspection
- **Protocol Specificity**: Must understand each protocol it handles

## Advanced Networking Concepts

### Network Access Control for Databases

Database systems like PostgreSQL implement network-level access controls through configuration files (e.g., pg_hba.conf in PostgreSQL). These controls allow administrators to:

- Restrict connections based on IP addresses and subnet masks
- Specify different authentication methods for different network segments
- Control which databases and users can connect from specific networks

This forms an important layer of security beyond application-level authentication.

> [https://www.postgresql.org/docs/current/auth-pg-hba-conf.html](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

### Protocol Analysis with Wireshark

Tools like Wireshark allow deep inspection of network traffic:

- **HTTPS and HTTP Analysis**: With proper key configuration, encrypted TLS traffic can be decrypted and analyzed
- **Stream Identification**: In protocols like HTTP/2, client streams use odd IDs while server streams use even IDs
- **Database Protocol Inspection**: Even proprietary protocols like MongoDB's wire protocol can be analyzed

**Wireshark on https and http**

**Filters:**
- ip.addr == 93.184.215.14
- tcp.port eq 443 and ip.addr eq 159.60.134.0

```sh
export SSLKEYLOGFILE=/Users/pravin.tripathi/tempkeys/key
source ~/.zshrc
sudo curl --insecure [https://nginx.com](https://nginx.com/)
```

**UI steps:**  
Wireshark -> preference -> TLS -> pre master secret log filename

![](image33.png)

> Client stream id 1 (odd) and server stream id (even)

### Protocol Ossification

Protocol ossification refers to the loss of flexibility and extensibility in network protocols over time:

- **Cause**: Primarily due to middleboxes that are sensitive to protocol formats
- **Impact**: Middleboxes may interfere with valid but unrecognized protocol messages
- **Principle Violation**: Represents a violation of the end-to-end principle
- **Example**: TLS 1.3 deployment faced challenges due to middleboxes expecting the format of older TLS versions

Protocol ossification is a significant consideration when designing new protocols or updating existing ones, as demonstrated by the development of QUIC. (**https://en.wikipedia.org/wiki/QUIC**)

### The Network Journey Behind Clicking a Link

When you click a hyperlink in your browser, a complex sequence of networking events occurs:

1. **DNS Resolution**: The domain name (e.g., pravin.dev) is resolved to an IP address
2. **TCP Connection**: A TCP connection is established to the server
3. **TLS Handshake**: For HTTPS sites, a secure TLS connection is negotiated
4. **Protocol Negotiation**: Using ALPN (Application-Layer Protocol Negotiation), the client and server agree on which protocol to use (e.g., HTTP/1.1, HTTP/2)
5. **Server Name Indication (SNI)**: The client indicates which hostname it's trying to reach (important for virtual hosting)
6. **HTTP Request**: The browser sends an HTTP request for the desired resource
7. **Redirection Chain**: Often, redirections occur (e.g., backend.husseinnasser.com → zen-mccarthy-34c0bb.netlify.app → udemy.com/backendcourse)
8. **Content Delivery**: The final server delivers the requested content
9. **Rendering**: The browser processes and renders the received content

This intricate process happens so quickly that users rarely notice the complexity involved.

## DNS: Finding the Right Address

When you click on `https://backend.husseinnasser.com`, the very first thing your browser needs to do is figure out where this website actually lives on the internet. This is where the Domain Name System (DNS) comes into play.

### The DNS Lookup Process

1. Your browser issues a DNS lookup for `backend.husseinnasser.com` to translate this human-readable domain name into an IP address that computers can use to communicate.

2. In this case, `backend.husseinnasser.com` is a CNAME (canonical name) record hosted on the DNS authoritative name server "enom" that points to a Netlify DNS record: `zen-mccarthy-34c0b.netlify.app` where the actual HTML file is hosted.

   > *Interesting note: This means you could directly visit `zen-mccarthy-34c0b.netlify.app` in your browser and it would also take you to the same content.*

3. The DNS query travels as a UDP datagram with a unique query ID, asking for the IP address of `backend.husseinnasser.com`.

### The DNS Resolution Journey

The DNS query follows a specific path:

1. **First Stop: DNS Resolver** - Your query first goes to a DNS recursor (resolver), which might be Google's `8.8.8.8` or Cloudflare's `1.1.1.1`, depending on your network configuration.

2. **Root DNS Servers** - The resolver asks the ROOT DNS servers for a `.com` top-level domain (TLD) server.

3. **TLD Server** - The resolver then queries a TLD server for the authoritative name server where `husseinnasser.com` is hosted, which returns one of the "enom" servers.

4. **Authoritative Name Server** - Finally, the resolver sends the DNS query for `backend.husseinnasser.com` to an "enom" server to get the IP address.

5. **CNAME Resolution** - The resolver discovers this is a CNAME pointing to `zen-mccarthy-34c0bb.netlify.app`, so it initiates a new DNS query for this domain, repeating the process until an actual IP address is discovered.

> **Technical Note:** There's a slight difference in terminology between a DNS lookup and DNS resolve. A DNS lookup in operating system terminology means it will try to find the IP address by checking local caches, hosts file, and finally making a network call to resolve. The DNS resolve is specifically the final part - going through the network to obtain the DNS information. The Linux function `getaddrinfo()` handles lookups and is [synchronous](https://medium.com/@hnasr/when-nodejs-i-o-blocks-327f8a36fbd4), which is significant for application performance.

You can observe this resolution process by using tools like `nslookup` or `dig` on the domain.

As part of the DNS query result for our example, we receive two A records with IPv4 addresses for the `zen-mccarthy` Netlify domain. Your browser will select one of these IPs (in this case `35.247.66.204`) to establish a TCP connection.

> **Load Balancing Note:** DNS can be configured to return multiple A record IP addresses. Round-robin DNS changes the order of the returned A records so that client connections can be distributed across different IP addresses. This prevents traditional clients (which always pick the first IP) from overloading a particular service. Advanced techniques can also be applied at the client level to implement client-side load balancing.

## TCP: Establishing a Connection

With the IP address in hand, your browser can now establish a Transmission Control Protocol (TCP) connection. This requires four pieces of information (often called a 4-tuple):

1. **Source IP** - Your machine's IP address (though this may change due to NAT)
2. **Source Port** - Any available port on your machine between 0 and 65535 (2¹⁶)
3. **Destination IP** - The server's IP address (`35.247.66.204` in our example)
4. **Destination Port** - Port 443, since the link specifies `https://`

> **NAT Note:** While the source IP might initially be your machine's private IP address, it changes to your gateway's public IP address before leaving your private network through a process called Network Address Translation (NAT). The source port also changes, and an entry is added to the NAT table to track this change so the gateway knows how to forward return packets back to your machine.

### The TCP Handshake

With these four pieces of information, the connection is established through a three-way handshake:

1. **SYN** - Your client sends a SYN (synchronize) TCP segment carried in an IP packet.
2. **SYN/ACK** - The server receives the SYN and replies with a SYN/ACK (synchronize/acknowledge), changing the destination IP to your client's IP.
3. **ACK** - Your client finalizes the handshake with an ACK (acknowledge).

At this point, we have an established connection!

> **Server Implementation Note:** Technically, the connection establishment on the server side is handled by the operating system kernel running on the server (`35.247.66.204` on port 443). When a backend application listens on a port/address, two queues are allocated: the SYN queue and the ACCEPT queue. The SYN queue holds incomplete connections (those that have received a SYN but not yet completed the handshake). Once the final ACK arrives from the client, the connection is created and moved to the Accept queue. The connection waits there until the backend application calls `accept()` to create a file descriptor and read the data.

## TLS: Securing the Connection

At this point, we have an established TCP connection, but any data sent over it would be in plain text, visible to anyone intercepting the traffic. This is where Transport Layer Security (TLS) comes in to ensure security (the "S" in HTTPS).

The main component of TLS is the handshake, which has three primary goals:

1. Exchange a symmetric encryption key
2. Negotiate application protocols
3. Authenticate the server

### The TLS Handshake

The process begins with:

1. **Client Hello** - Your browser sends a TLS Client Hello message to initiate the TLS handshake, requesting session encryption and proposing supported protocols (like HTTP/1.1 and HTTP/2).
2. **Server Hello** - The server replies with a Server Hello message to continue the TLS handshake.

The Client Hello message includes several TLS extensions. Two particularly important ones are:

### ALPN (Application Layer Protocol Negotiation)

ALPN is a TLS extension that indicates which application protocols the client supports. In our example, the browser proposes both HTTP/1.1 and HTTP/2 (abbreviated as "h1" and "h2"). The highest protocol supported by both client and server is typically selected - in this case, HTTP/2.

### SNI (Server Name Indication)

Perhaps the most critical piece of the TLS handshake is SNI (Server Name Indication). This extension tells the server which domain the client wants to access. This is crucial because one IP address can host thousands of websites, and the server needs to know which specific site the client is requesting to serve the correct TLS certificate.

In our example, the client SNI is set to `backend.husseinnasser.com`. When the Netlify server receives this, it knows exactly which certificate to provide for authentication - in this case, a Let's Encrypt certificate generated specifically for this domain when it was linked with Netlify.

The server then sends the Server Hello to complete the TLS handshake, including:
- The certificate for `backend.husseinnasser.com`
- The cipher parameters
- The selected application protocol (HTTP/2)

At this point, both client and server have the symmetric key for encryption and are ready to encrypt and exchange HTTP requests.

> **TLS Version Note:** The client typically proposes multiple TLS protocol versions (like TLS 1.3 and TLS 1.2) without knowing which one the server will accept. If the server chooses TLS 1.3, the handshake completes in one round trip; otherwise, it takes two round trips with TLS 1.2. In our example, we assume TLS 1.3 is used since Netlify supports it.
>
> With TLS 1.3, the certificate and other parameters are encrypted because the server already has the symmetric key. This differs from TLS 1.2, where the certificate is sent in plain text in the Server Hello because the encryption key isn't created until the second round trip. More details on this can be found in [this article](https://medium.com/@hnasr/the-many-configurations-of-https-4fa005a456ad).


**Allow hosting multiple host on same IP address.**  
It's time to replace tcp in the datacenter -> HOMA
{% include presentation.html %}

## HTTP/2: Making the Request

Now that we have an encrypted session on a TCP connection, the browser can send the HTTP GET request to fetch the webpage. Since the negotiated protocol is HTTP/2, the client needs to create an HTTP/2 stream for the request.

Being a new connection, the browser creates a new stream (stream 1) and sends an HTTP request with:
- Method: GET
- Path: / (since nothing follows `backend.husseinnasser.com` in the URL)
- Protocol version: HTTP/2
- Various HTTP headers

### The Critical Host Header

One of the most important HTTP headers is the `Host` header. Without it, the entire system would break down. The Host header was initially added as an optional feature in HTTP/1.0 to solve challenges for webmasters, such as:

- Enabling multiple websites to be hosted on a single IP address
- Supporting proxy server configurations

The Host header became mandatory in HTTP/1.1 and later protocols for these reasons.

In our example with Netlify, the IP address `35.247.66.204` hosts thousands of websites. When clients connect to this IP, the Host header (set to `backend.husseinnasser.com`) tells Netlify's servers exactly which website content to serve.

The client might also send cookies with the GET request, which the server can use to identify the user.

In proxy configurations, the client's destination IP is the proxy itself, so the Host header provides the necessary information for the proxy to know which website the client actually wants to access.

The server receives the GET request on stream 1, processes it by checking the Host header, and fetches the default page (index.html). Based on whether cookies are present, the server might return different HTTP responses on the same stream.

> **Connection Reuse Note:** It's important to mention that HTTP requests to the same domain will typically reuse an existing connection when possible to avoid the cost of creating and encrypting new connections. HTTP/2 supports request multiplexing, allowing clients to send multiple concurrent requests on the same connection using different streams. In HTTP/1.1, however, one connection can only serve one request at a time.

## The Final Step: Redirection to Udemy

In our example, the client finally receives an HTTP response containing the index.html page, which consists of the following HTML:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="refresh" content="0; URL=https://www.udemy.com/course/fundamentals-of-backend-communications-and-protocols/?couponCode=BACKEND10V2" />
  <title>Fundamentals of Backend Communication Design Patterns and Protocols</title>
</head>
<body>
  <h1>Redirecting to udemy ...</h1>
  <h2>Enjoy my Fundamentals of Backend Engineering Course</h2>
</body>
</html>
```

The key element here is the meta tag with `http-equiv="refresh"` and `content="0; URL=..."`. This instructs the browser to immediately redirect to the specified Udemy URL. The entire process we've just examined repeats as if the user had clicked a new link:

1. DNS lookup for `udemy.com` to find its IP address
2. TCP connection establishment
3. TLS handshake
4. HTTP protocol negotiation
5. HTTP/2 stream creation
6. New GET request with the path `/course/fundamentals-of-backend-communications-and-protocols/?couponCode=BACKEND10V2`
7. If the user is logged into Udemy, cookies will be sent
8. Udemy's server responds with the course page based on the user's sign-in state

The `couponCode` query parameter at the end of the URL is particularly noteworthy. The author updates this coupon monthly and pushes changes to GitHub, which triggers a Netlify rebuild. Anyone visiting `backend.husseinnasser.com` gets the latest coupon. If the author decides to switch course providers in the future, they can simply update the URL in index.html while the original link remains unchanged.

The goal of this article is to shed light on the art of software engineering. A lot of work has been put in by great engineers who designed and implemented the communication protocols that power everything on the Internet.
As a fellow engineer, I feel that this work is often taken for granted and rarely acknowledged. Understanding how protocols work is the first step to contributing to the evolution of network engineering and potentially making better protocols. You've seen what it takes to achieve a simple task such as clicking a link, which raises the question: can we make it better?
Understanding these underlying processes not only satisfies curiosity but also provides valuable insights for web developers, network engineers, and anyone interested in how the internet functions at a fundamental level.
Next time you click a link, you'll have a deeper appreciation for the complex sequence of events happening behind the scenes to bring that content to your screen.

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*