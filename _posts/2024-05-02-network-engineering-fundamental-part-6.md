---
title: "Fundamentals of Network Engineering: Network Intermediaries - Proxies and Load Balancers - Part 6"
author: pravin_tripathi
date: 2024-05-02 00:00:00 +0530
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

## Proxy Systems: Intermediaries in Network Communication
Networks often involve intermediaries that route or manage traffic. This article distinguishes between client-side proxies and server-side reverse proxies, and then delves into the specific function of load balancers at different network layers.

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

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*