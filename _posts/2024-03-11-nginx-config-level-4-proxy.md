---
title: "NGINX: NGINX config for Level 4 proxy"
author: pravin_tripathi
date: 2024-03-11 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-config-level-4-proxy/
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

```config
stream {
    
    upstream allbackend {
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
        server 127.0.0.1:5555;
    }
    

    server {
          listen 80;
          proxy_pass allbackend;
 
     }

}

events { } 
```

This configuration is for **Nginx stream module**, which is used for handling TCP or UDP traffic (rather than HTTP). It is commonly used for load balancing and proxying for non-HTTP protocols like database connections, email servers, or any other type of stream-based protocol.

### Breakdown of the Configuration:

### 1. **Stream Block**

```
stream {
    ...
}

```

- The `stream` block is used to configure the stream module in Nginx, which handles TCP and UDP connections. This is different from the `http` block used for web traffic.

---

### 2. **Upstream Definition (`allbackend`)**

```
upstream allbackend {
    server 127.0.0.1:2222;
    server 127.0.0.1:3333;
    server 127.0.0.1:4444;
    server 127.0.0.1:5555;
}

```

- The `upstream` block defines a group of backend servers. In this case, the backend servers are all running on the same machine (`127.0.0.1` or `localhost`), but they are listening on different ports (`2222`, `3333`, `4444`, `5555`).
- This block allows Nginx to distribute incoming stream (TCP/UDP) traffic among these backend servers for load balancing.

---

### 3. **Server Block (Stream Proxying)**

```
server {
    listen 80;
    proxy_pass allbackend;
}

```

- This server block listens on **port 80** for incoming **TCP connections**.
- The `proxy_pass allbackend;` directive tells Nginx to forward these incoming connections to the backend servers defined in the `allbackend` upstream block.
    - Nginx will load balance between the four backend servers (`127.0.0.1:2222`, `127.0.0.1:3333`, `127.0.0.1:4444`, and `127.0.0.1:5555`) for any incoming TCP traffic on port 80.
- This configuration is useful for scenarios where you want to distribute TCP traffic (such as database, game server, or custom protocol traffic) to multiple backend servers.

---

### 4. **Events Block**

```
events { }

```

- The `events` block is empty but required for Nginx to function. It manages worker processes and connection handling, similar to the HTTP block, but it's not used in the `stream` module itself.

---

### **Summary of How it Works:**

- **Upstream Block**: Defines four backend servers (running on different ports) where the stream traffic will be forwarded.
- **Server Block**: Listens for **TCP connections on port 80** and forwards them to the backend servers defined in the `allbackend` upstream block, distributing the traffic across those servers.
- This is typically used for load balancing and proxying **non-HTTP traffic**, such as TCP-based protocols (e.g., MySQL, PostgreSQL, or custom TCP services).

### Use Case Example:

- Suppose you have a service running on multiple instances (e.g., a game server, or a database), and you want Nginx to act as a load balancer. It will listen for incoming TCP connections on port 80 and forward those connections to one of the four backend servers (`127.0.0.1:2222`, `127.0.0.1:3333`, etc.) based on the load balancing algorithm.

This configuration is quite simple but effective for handling TCP stream traffic!

[Back to Parent Page]({{ page.parent }})