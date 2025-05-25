---
title: "Fundamentals of Network Engineering: Securing Network Communications with TLS - Part 8"
author: pravin_tripathi
date: 2024-05-18 00:00:00 +0530
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

> With TLS 1.3, the certificate and other parameters are encrypted because the server already has the symmetric key. This differs from TLS 1.2, where the certificate is sent in plain text in the Server Hello because the encryption key isn't created until the second round trip. More details on this can be found in [this article](https://medium.com/@hnasr/the-many-configurations-of-https-4fa005a456ad).

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*