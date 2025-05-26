---
title: "NGINX: Http 2 with Secure Nginx"
author: pravin_tripathi
date: 2024-03-06 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/http-2-with-secure-nginx/
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
http {
    
    upstream allbackend {
        #ip_hash;
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
        server 127.0.0.1:5555;
    }
    
    upstream app1backend {
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
    }

    upstream app2backend {
        server 127.0.0.1:4444;
        server 127.0.0.1:5555;
    }

    server {
          listen 80;
          listen 443 ssl http2;

          ssl_certificate /etc/letsencrypt/live/nginxtest.ddns.net/fullchain.pem;
          
          ssl_certificate_key /etc/letsencrypt/live/nginxtest.ddns.net/privkey.pem;

          ssl_protocols TLSv1.3;

          location / {

              proxy_pass http://allbackend/;
           }

         location /app1 { 
              proxy_pass http://app1backend/;
          }
        
        location /app2 {
              proxy_pass http://app2backend/;
         }
         
        location /admin {
          return 403;
        }          
     }

}

events { } 
```

This Nginx configuration sets up a web server with SSL support and reverse proxying. It also includes some load balancing between multiple backend servers for different applications, and blocks access to `/admin` with a `403 Forbidden` response.

### Breakdown of the Configuration:

---

### 1. **HTTP Block**

```
http {
    ...
}

```

This is the main block that contains the HTTP server configuration. It encompasses the settings for handling HTTP and HTTPS traffic, load balancing, proxying requests, and other HTTP-related configurations.

---

### 2. **Upstream Definitions**

```
upstream allbackend {
    server 127.0.0.1:2222;
    server 127.0.0.1:3333;
    server 127.0.0.1:4444;
    server 127.0.0.1:5555;
}

```

The `upstream` block defines a group of backend servers. In this case, four backend servers are defined for load balancing:

- These backend servers are all located on `127.0.0.1` (localhost) but are running on different ports: `2222`, `3333`, `4444`, and `5555`.
- Requests that hit this upstream will be forwarded to one of these backend servers, with Nginx balancing the load (using round-robin by default).

### `upstream app1backend`

```
upstream app1backend {
    server 127.0.0.1:2222;
    server 127.0.0.1:3333;
}

```

- This defines another upstream group specifically for `app1`. It uses only two backend servers: `127.0.0.1:2222` and `127.0.0.1:3333`.

### `upstream app2backend`

```
upstream app2backend {
    server 127.0.0.1:4444;
    server 127.0.0.1:5555;
}

```

- Similarly, this defines a group of two backend servers specifically for `app2`: `127.0.0.1:4444` and `127.0.0.1:5555`.

---

### 3. **Server Block (HTTPS and HTTP)**

```
server {
    listen 80;
    listen 443 ssl http2;

    ssl_certificate /etc/letsencrypt/live/nginxtest.ddns.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nginxtest.ddns.net/privkey.pem;

    ssl_protocols TLSv1.3;
    ...
}

```

This server block listens on two ports:

- **Port 80 (HTTP)**: Handles non-SSL traffic. Nginx will serve requests on this port but it is common to redirect these to HTTPS for security reasons (though this specific config doesn't do a redirect from HTTP to HTTPS).
- **Port 443 (HTTPS)**: Handles secure SSL traffic. It uses the **SSL certificate** (`ssl_certificate`) and **private key** (`ssl_certificate_key`) to encrypt the connection using the TLS protocol.
    - `ssl_protocols TLSv1.3;`: This ensures that only **TLSv1.3** is used for secure connections, which is the latest and most secure version of the TLS protocol.

### `ssl_certificate` and `ssl_certificate_key`

- These directives specify the path to the SSL certificate and private key for HTTPS. The SSL certificate is typically issued by a certificate authority (CA), and in this case, the paths indicate that it's using **Let's Encrypt** certificates.

---

### 4. **Location Blocks (Proxying Traffic)**

Each `location` block in Nginx defines how to handle different types of requests based on their URL path.

### `location /`

```
location / {
    proxy_pass http://allbackend/;
}

```

- The root URL (`/`) of the server will proxy requests to the upstream servers defined in `allbackend` (the four backend servers on ports `2222`, `3333`, `4444`, `5555`).
- Nginx will load balance between these backend servers and forward the requests accordingly.

### `location /app1`

```
location /app1 {
    proxy_pass http://app1backend/;
}

```

- Requests to `/app1` will be forwarded to the backend servers defined in the `app1backend` upstream. This is a smaller group of two backend servers (`127.0.0.1:2222` and `127.0.0.1:3333`).

### `location /app2`

```
location /app2 {
    proxy_pass http://app2backend/;
}

```

- Similarly, requests to `/app2` will be forwarded to the backend servers defined in the `app2backend` upstream, which are `127.0.0.1:4444` and `127.0.0.1:5555`.

### `location /admin`

```
location /admin {
    return 403;
}

```

- Requests to `/admin` will be **blocked** and a `403 Forbidden` error will be returned.
- This is a common practice to prevent unauthorized access to sensitive areas of a website or application (like an admin panel).

---

### 5. **Events Block**

```
events { }

```

- The `events` block is required but not used in this particular configuration.
- It typically contains directives that control Nginx’s event-driven model, such as worker processes or connection handling. In this case, it’s left empty, meaning Nginx will use the default settings.

---

### **Summary of How it Works:**

1. **SSL and HTTP**:
    - The server listens on both **port 80 (HTTP)** and **port 443 (HTTPS)**. Traffic on port 443 is encrypted with SSL/TLS using the specified certificate.
    - It only allows **TLSv1.3** for secure connections.
2. **Load Balancing**:
    - The server balances traffic across different sets of backend servers:
        - Requests to `/` are forwarded to all backend servers (`allbackend`).
        - Requests to `/app1` are forwarded to a subset of backend servers (`app1backend`).
        - Requests to `/app2` are forwarded to another subset (`app2backend`).
3. **Blocking `/admin`**:
    - Any requests to `/admin` will be denied with a `403 Forbidden` response.
4. **SSL Configuration**:
    - The server uses a **Let’s Encrypt SSL certificate** to encrypt traffic and ensure secure communication via **TLSv1.3**.

This setup is ideal for scenarios where you want to:

- Serve multiple applications or services using a single Nginx server.
- Use SSL for secure communication.
- Balance load across multiple backend servers to improve performance and reliability.

[Back to Parent Page]({{ page.parent }})