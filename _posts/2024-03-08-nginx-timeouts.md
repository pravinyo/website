---
title: "NGINX: Nginx timeouts"
author: pravin_tripathi
date: 2024-03-08 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-timeouts/
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

# Nginx Timeouts

## Frontend Timeouts

- client_header_timeout

![image.png](nginx-timeouts/image.png)

https://www.cloudflare.com/learning/ddos/ddos-attack-tools/slowloris/

- client_body_timeout

![image.png](nginx-timeouts/image%201.png)

- send_timeout

![image.png](nginx-timeouts/image%202.png)

- keepalive_timeout

![image.png](nginx-timeouts/image%203.png)

- lingering_timeout

![image.png](nginx-timeouts/image%204.png)

- resolver_timeout

![image.png](nginx-timeouts/image%205.png)

## Backend Timeouts

- proxy_connect_timeout

![image.png](nginx-timeouts/image%206.png)

- proxy_send_timeout

![image.png](nginx-timeouts/image%207.png)

- proxy_read_timeout

![image.png](nginx-timeouts/image%208.png)

- keepalive_timeout

![image.png](nginx-timeouts/image%209.png)

- proxy_next_upstream_timeout

![image.png](nginx-timeouts/image%2010.png)

[Back to Parent Page]({{ page.parent }})