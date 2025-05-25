---
title: "Fundamentals of Network Engineering: The Journey of a Click - From DNS to Redirection - Part 7"
author: pravin_tripathi
date: 2024-05-10 00:00:00 +0530
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

## The Network Journey Behind Clicking a Link

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


*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*