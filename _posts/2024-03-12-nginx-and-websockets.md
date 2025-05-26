---
title: "NGINX: NGINX and WebSockets"
author: pravin_tripathi
date: 2024-03-12 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-and-websockets/
parent: /nginx-understanding-and-deployment/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, network-engineering, nginx]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

## Introduction to WebSockets

**WebSockets** are a communication protocol that provides full-duplex, bi-directional communication channels over a single, long-lived connection between a client (usually a web browser) and a server. Unlike traditional HTTP requests, which are short-lived and require a new connection for each request/response cycle, WebSockets enable continuous, real-time communication without the need to constantly re-establish connections.

Here’s a brief overview of how WebSockets work:

1. **Establishing a Connection**: The WebSocket connection begins as an HTTP handshake, where the client (typically a web browser) sends an HTTP request to the server asking to upgrade the connection to a WebSocket. If the server supports WebSockets, it responds with a successful handshake and switches to the WebSocket protocol.
2. **Full-Duplex Communication**: After the handshake, the connection remains open, allowing both the client and server to send messages at any time without needing to wait for a request. This enables two-way communication, making WebSockets ideal for applications that require real-time updates, like chat applications, online games, and live data feeds.
3. **Efficiency**: WebSockets reduce overhead by eliminating the need for re-establishing connections for each message, unlike HTTP where each request requires a new connection setup. This makes WebSockets much more efficient for frequent or low-latency communication.
4. **Use Cases**: WebSockets are widely used in applications that require real-time interaction, such as:
    - Online chat apps
    - Live sports scores
    - Collaborative tools
    - Financial tickers
    - Multiplayer online games
5. **Protocol**: WebSocket communication happens over the `ws://` (non-secure) or `wss://` (secure) protocols, similar to how HTTP uses `http://` and `https://`.

### Example Flow:

- **Client**: Sends an HTTP request with an "Upgrade: websocket" header to initiate the handshake.
- **Server**: Responds with a 101 status code and switches protocols to WebSocket.
- **Client and Server**: Exchange messages through the open WebSocket connection until either party closes it.

### Benefits:

- Real-time, low-latency communication
- Reduced network overhead
- Ideal for interactive, data-intensive applications

WebSockets are particularly powerful for web applications that need real-time communication between the server and the client without constantly polling the server for updates.

Http 1.0

![image.png](nginx-and-websockets/image.png)

Http 1.1

![image.png](nginx-and-websockets/image%201.png)

Websocket

![image.png](nginx-and-websockets/image%202.png)

![image.png](nginx-and-websockets/image%203.png)

![image.png](nginx-and-websockets/image%204.png)

## Layer 4 vs Layer 7 WebSockets Proxying

- In Layer 4 OSI model we see TCP/IP content
    - Connections, Ports, IP addresses.
    - Content remains encrypted (if unencrypted it is not inspected)
- In Layer 7 OSI Model we see all what's below
    - Layer 4 + Application layer content
    - Content is decrypted (TLS termination)
    - We can read headers, paths, urls etc.
- Layer 4 Proxying on WebSockets is done as a tunnel
- NGINX intercepts the SYN for a connection and creates another connection on the backend
- Any data sent on the frontend connection is tunneled to the backend connection
- The backend connection remains private and dedicated to this client.

![image.png](nginx-and-websockets/image%205.png)

![image.png](nginx-and-websockets/image%206.png)

![image.png](nginx-and-websockets/image%207.png)

![image.png](nginx-and-websockets/image%208.png)

[Back to Parent Page]({{ page.parent }})