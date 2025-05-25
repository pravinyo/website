---
title: "Fundamentals of Network Engineering: Advanced Tools, Concepts, and Conclusion - Part 9"
author: pravin_tripathi
date: 2024-05-25 00:00:00 +0530
readtime: true
media_subpath: /assets/img/network-engineering-fundamental/
file_document_path: "/assets/document/attachment/network-engineering-fundamental/2210.00714v2.pdf"
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, network-engineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

Finally, let's touch on a few more advanced topics, including database access control, network analysis tools, and challenges in protocol evolution, before concluding with a reflection on the importance of understanding these fundamentals.

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


## Allow hosting multiple host on same IP address.
It's time to replace tcp in the datacenter -> HOMA
{% include presentation.html %}

## Conclusion
The goal of this article is to shed light on the art of software engineering. A lot of work has been put in by great engineers who designed and implemented the communication protocols that power everything on the Internet.
As a fellow engineer, I feel that this work is often taken for granted and rarely acknowledged. Understanding how protocols work is the first step to contributing to the evolution of network engineering and potentially making better protocols. You've seen what it takes to achieve a simple task such as clicking a link, which raises the question: can we make it better?
Understanding these underlying processes not only satisfies curiosity but also provides valuable insights for web developers, network engineers, and anyone interested in how the internet functions at a fundamental level.
Next time you click a link, you'll have a deeper appreciation for the complex sequence of events happening behind the scenes to bring that content to your screen.

*This blog post was compiled from my notes on a Networking Fundamentals course. I hope it helps clarify these essential concepts for you!*