---
title: "NGINX: Threading and Connections"
author: pravin_tripathi
date: 2024-03-03 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-threading-and-connections/
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

Building a Backend application requires a communication protocol, a port to bind to, and process to serve requests and produce responses. When using TCP in particular as the transport protocol, a stateful connection is created for the client who wishes to connect to the backend. Before this can happen the backend application has to actively accept a connection, otherwise connections will remain in the operating system backlog buffer which will eventually fill up. Reading the TCP stream from a connection is also the job of the backend application, this moves raw stream bytes from the OS buffer up to the backend application.

In this post I explore five architecture patterns on how threads and connections are managed. Before I do that, let us clear up some definitions so we are on the same page.

Listener — When the backend applications listens on a particular IP and port it creates a socket. A socket is not a connection but a place to connect to. Just like a wall socket, where you can plug in connections. Many connections can be connected to the socket. The listener is the process where the socket lives.

Acceptor — With a socket in hand, the backend can call the OS function *accept* passing this socket to accept any available connections on this socket. The *accept* function returns a [file descriptor](https://en.wikipedia.org/wiki/File_descriptor) representing the connection. Connections have to be actively accepted by the application in order to serve clients. Otherwise the connections will remain in the OS *accept* queue unused. The acceptor is the thread or process that calls the *accept* function.

Reader — You can also call it worker too, this is where the work happens. When we have a connection, the backend application is also responsible to read data sent to this connection otherwise the buffer allocated by the OS for this connection will fill up and client won’t be able to send any more data. I call the process that reads the TCP stream on the connection the Reader. The reader passes in the file descriptor (connection) to read the stream and act on it.

TCP Stream vs Requests — TCP is a streaming protocol. Let us translate what that means for our HTTP web server. When you send a GET request through Axios from the frontend, Axios will create a TCP connection (if one doesn’t exist) and build out the HTTP request consisting of the method, protocol version, headers , URL parmaters etc. The HTTP request is well defined it has a start and an end. But guess what? the TCP stream is just raw bytes of data, so the reader is responsable to read all the stream and “look” for requests, oh here is a start of the request, let me continue reading, ok I see headers, URL, and yeah that is the end of the request. This collection of bytes is now translates to a logical request (sometimes called message because we like to confuse people) that request is delivered to the application layer 7 for processing. So it is not as easy we all think. This is true for any layer 7 protocol that uses TCP, HTTP/2, gRPC, SSH. This parsing can take a toll on whatever thread does it.

With that out of the way, let us go through the five different architectures to manage threads and connections. Keep in mind that these are patterns I observed personally in the wild from my 18 year career. Does that mean nothing else exist, of course not.

### Single Threaded Architecture

A simple, elegant and easy to understand architecture. Where the backend application consists of a single thread that acts as a listener, so it binds on an IP:Port, it accepts connections on the socket, and it also reads the TCP stream from all connections it accepts. While this single thread can quickly become overwhelmed, some backend engineers find this architecture attractive to use as it keeps their application simple. They opt to instead scale their backend horizontally by spinning up multiple instances of this simple application as oppose of using multiple threads in the backend. [NodeJS](https://www.youtube.com/watch?v=gMtchRodC2I) for instance uses this model and as a result any application you build using NodeJS.

![](https://img-b.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-56-41-6adb13d1e4b4bef551d02e40fd5d5398.png)

### Multiple Threads Single Acceptor Architecture

Multi threading can be used to build performant backend applications that takes advantage of all CPU cores. The price you pay as an engineer is complexity (nothing is free) but sometimes its worth it.

In this architecture, you still have a single listener thread and that same thread usually also accepts connections. The difference here is each accepted connection is handed over to another thread. Once a thread has a connection it will call read on the connection file descriptor using any IO paradigm is popular these days ([io_uring](https://man.archlinux.org/man/io_uring.7.en) seems to be the jam in 2022). [memcached](https://memcached.org/) use this architecture.

I used to think that for each new accepted connection you can just spin up a new thread and hand it over, but that turns out to be a bad idea. Pretty soon you will run out of memory and if even you don’t, the context-switching between the different threads in a limited number of cores will significantly hinder performance.

A more balanced approach in memcached is to specify a max number of threads and threads will share multiple connections. A good rule-of-thumb for max number of threads is equal to the number of CPU cores you have and assuming you don’t have tons of other programs running on your rig, that should be good start. This ensures the thread sticks to a core and keeps context-switching to a minimum, reusing that beautiful L2/L1 cache.

The limitation of this approach is we have a single thread accepting all connections so if there are a flood of connection requests, the thread can lag behind on accepting connections which will lead to clients getting high latency times. It is important to note here that the SYN/SYN-ACK/ACK three way TCP handshake has already happened and performed by the OS, this is just the backend application moving the connection from the OS *accept* queue to its own buffer. If the backend application can’t accept connections fast enough and empty up the *accept* queue, no more clients can connect to the host (the three way handshake will fail or timeout).

Another limitation of this is lack of load balancing. Some threads might end up processing a greedy client connection with many requests while other threads connections barely send any requests. This creates hot-spots where one thread is overwhelmed while other threads aren’t; causing further latencies on connections that happened to land on the busy thread.

Watch my [YouTube coverage on memcached](https://youtu.be/NCePGsRZFus) to learn more about this in memory store.

![](https://img-b.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-56-42-04f195205978982cb66e8a0930324cb9.png)

### Multiple Threads Multiple Acceptors Architecture

Just like the previous architecture, we will use a single listener thread to create the socket. However we place the socket in a shared memory where other threads can access it. The listener thread creates multiple worker threads and pass it the shared socket object. Each thread will call *accept* on the socket object to accept connections. The connection resulting from the *accept* call becomes the thread’s responsibility, the thread in this case is both the acceptor and the reader. This model disperses the responsibility of connection management to local threads. [NGINX](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/amp/) prior to version 1.9.1 used to use this particular architecture by default.

The limitation of this architecture is all threads compete to accept connections on a single shared socket object which requires a mutex on the *accept* queue. This causes threads to be serialized while accepting connections and may lead to blocking and high latencies. While sure faster than having a single thread accept all connections, it is not as fast as it could be, read along to find out more tricks. Also the problem of load balancing also is still present in this architecture.

![](https://img-b.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-56-42-4cfc9f56d0f83573e24948288e15527b.png)

### Multiple Threads with Message-based Load Balancing Architecture

This architecture is very interesting to me, I learned about it while reviewing the [Homa](https://web.stanford.edu/~ouster/cgi-bin/papers/replaceTcp.pdf) paper, a protocol attempts to replace TCP for the data center. It is similar to memcached’s architecture but instead of handing over the connection (raw file descriptors) to the threads, the listener thread accepts, reads and parses logical messages (requests) and deliver clean cut requests to the threads where it is processed. [RAMCloud](https://dl.acm.org/doi/10.1145/2806887) uses this architecture and it's very advantageous to load balancing requests among threads, the listener thread is aware of what the worker threads are doing and can distribute the load evenly.

The problem with this architecture is the listener thread becomes a bottle-neck as not only it is responsible to have all connections but it must also call *read* on all of them.

![](https://img-b.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-56-42-a2380605b2d2828acb94b60b8d592a21.png)

### Multiple Threads with Socket Sharding (SO_REUSEPORT)

One sort of limitation we have with binding a socket is only one process can listen on a given port. No two processes can listen on the same port/IP, the OS won’t allow it (by default). Recently the OS relaxed this constraint if the socket listener specifies the socket option (SO_REUSEPORT) this allows multiple processes to listen on the same port and both processes will get a different handle to the socket while both point to the same address/port. My guess here is the OS will create an *accept* queue for each of the bound socket and will distribute connections destined to the port/address to the multiple *accept* queues. The *accept* calls won’t be serialized or blocked in this case as each has its own queue. NGINX 1.9. 1 started supporting socket sharding so does [Envoy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto) and HAProxy.

Just like everything in software engineering, I’m pretty sure this one has its own limitations, I just didn’t find it yet. I will keep locking and update this article when I do.

The cool thing about all this is you can mix and match architectures. For example, I never seen this one yet, but mixing socket sharding with message based load balancing sounds like a beast of an architecture.

![](https://img-b.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-56-42-fd13469d9e356bb6c6be4c88f48d7da9.png)

[Back to Parent Page]({{ page.parent }})