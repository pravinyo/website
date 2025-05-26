---
title: "NGINX: NGINX config for Level 7 proxy"
author: pravin_tripathi
date: 2024-03-10 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-config-level-7-proxy/
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

### 1. **HTTP Block**

```
http {
    ...
}

```

The `http` block is the main container for HTTP server settings. All configurations related to serving HTTP requests (such as reverse proxying, load balancing, etc.) are inside this block.

### 2. **Upstream Servers**

In Nginx, an **upstream** is used to define a group of backend servers. Nginx can distribute requests to these servers, which helps with load balancing.

### `upstream allbackend`

```
upstream allbackend {
    server 127.0.0.1:2222;
    server 127.0.0.1:3333;
    server 127.0.0.1:4444;
    server 127.0.0.1:5555;
}

```

This defines a set of backend servers that Nginx will proxy traffic to. In this case, there are four servers running on the same machine (localhost, `127.0.0.1`) but on different ports (`2222`, `3333`, `4444`, and `5555`).

- **ip_hash** (commented out) is an optional directive that would make sure requests from the same client (IP) are always sent to the same backend server. This is useful for session persistence, but it's not enabled in this config since it's commented out.

### `upstream app1backend`

```
upstream app1backend {
    server 127.0.0.1:2222;
    server 127.0.0.1:3333;
}

```

This upstream defines a group of two backend servers for "app1", running on ports `2222` and `3333`.

### `upstream app2backend`

```
upstream app2backend {
    server 127.0.0.1:4444;
    server 127.0.0.1:5555;
}

```

This upstream defines a group of two backend servers for "app2", running on ports `4444` and `5555`.

### 3. **Server Block**

```
server {
    listen 80;
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

```

- **listen 80**: This configures the server to listen on port 80, the default HTTP port.
- **location /**: This defines how to handle requests to the root of the website (`/`):
    - Requests to the root (e.g., `http://yourdomain.com/`) will be forwarded to the upstream `allbackend`. This means the request will be passed to one of the backend servers defined in the `allbackend` upstream block (ports `2222`, `3333`, `4444`, `5555`).
- **location /app1**: This handles requests to `/app1`:
    - Requests to `http://yourdomain.com/app1` will be forwarded to one of the backend servers defined in the `app1backend` upstream block (ports `2222`, `3333`).
- **location /app2**: This handles requests to `/app2`:
    - Requests to `http://yourdomain.com/app2` will be forwarded to one of the backend servers defined in the `app2backend` upstream block (ports `4444`, `5555`).
- **location /admin**: This handles requests to `/admin`:
    - Requests to `http://yourdomain.com/admin` will receive a `403 Forbidden` error. This is often used to block access to sensitive areas of a site, such as an admin panel.

### 4. **Events Block**

```
events { }

```

This block is used to configure event-related settings in Nginx, such as worker processes. In this case, it's empty, meaning it will use the default settings.

### Summary of How it Works:

- **Load balancing**: Nginx will balance requests across multiple backend servers using the `upstream` directives.
- **Root and application-specific routing**:
    - The root (`/`) routes traffic to a larger pool of backend servers (`allbackend`).
    - The `/app1` and `/app2` locations route traffic to separate pools of backend servers (`app1backend` and `app2backend` respectively).
    - Access to `/admin` is blocked with a `403 Forbidden` response.

[Back to Parent Page]({{ page.parent }})