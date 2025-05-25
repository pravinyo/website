---
title: "Fundamentals of Network Engineering: Managing Connections and Performance - Part 5"
author: pravin_tripathi
date: 2024-04-25 00:00:00 +0530
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

# The Cost of Connections
Establishing and managing connections comes with inherent costs. This article explores why connections are expensive, how connection pooling mitigates this, different loading paradigms, techniques for faster connection setup, and the phenomenon of Head-of-Line blocking.

## Why Connections Are Expensive

Establishing TCP connections incurs significant costs:

- **Handshake Overhead**: The three-way TCP handshake takes time
- **Distance Penalty**: Greater physical distance between endpoints increases latency
- **Slow Start**: TCP doesn't reach full throughput immediately
- **Control Mechanisms**: Congestion and flow control limit performance
- **Algorithm Impact**: Nagle's algorithm and delayed acknowledgments add latency
- **Termination Cost**: Connection teardown is also resource-intensive

## Connection Pooling: A Critical Optimization

To mitigate connection costs, most database backends and reverse proxies implement connection pooling:

- Pre-establish multiple TCP connections to backend services
- Keep connections open and ready for use
- Serve incoming requests using these "warm" connections
- Allow slow start to complete ahead of time
- Close connections only when absolutely necessary

This approach dramatically improves response times for clients.

## Eager vs. Lazy Loading

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

## TCP Fast Open (TFO)

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

## TCP Head-of-Line (HOL) Blocking in HTTP/1 and HTTP/2

TCP's reliable delivery mechanism can create performance bottlenecks:

- Lost packets block all subsequent data until retransmission succeeds
- In HTTP/1.1, browsers open multiple TCP connections to work around this limitation
- HTTP/2 uses a single TCP connection with multiple streams
- While HTTP/2 avoids application-level HOL blocking, it's still vulnerable to TCP-level HOL blocking
- HTTP/3 addresses this by using QUIC, which is built on UDP

![](image24.png)
![](image29.png)

[previous part](../network-engineering-fundamental-part-4)


[continue to next part](../network-engineering-fundamental-part-6)

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*