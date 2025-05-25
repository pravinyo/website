---
title: "Fundamentals of Network Engineering: Essential Network Protocols - ICMP and UDP - Part 2"
author: pravin_tripathi
date: 2024-04-10 00:00:00 +0530
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

## Network Protocols: The Language of the Internet
Network protocols are the language of the Internet. Let's look at two fundamental ones: ICMP for diagnostics and UDP for simple, fast transmissions.

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

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*