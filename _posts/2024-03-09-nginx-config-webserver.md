---
title: "NGINX: NGINX config webserver"
author: pravin_tripathi
date: 2024-03-09 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginx-config-webserver/
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

    # simple web static files server 
    server {
        listen 8080;
        # location of main index.html file
        root /Users/HusseinNasser/nginxcourse;
       
        # serving an entirely different directory than root
        location /images {
           
            root /Users/HusseinNasser/;
        }
       
        # excluding any file having an extension/occurence of the letters jpg at the end
        location ~ .jpg$ {

             return 403;
        }
   
    }

   # web server proxy for our server above 
   server {
           listen 8888;
            
            
           location / {
              
                proxy_pass http://localhost:8080/;
           }
            

           location /img {

                proxy_pass http://localhost:8080/images/;
           }

    }

}

# due to some reasons, NGINX requires this block!
events { }
```

This Nginx configuration sets up two different web servers, one serving static files on port `8080`, and the other acting as a reverse proxy on port `8888`. Let's go through it step-by-step:

### 1. **HTTP Block**

```
http {
    ...
}

```

As before, this is the outer block that contains all the HTTP server configurations. It encompasses the entire configuration for web serving and proxying.

---

### 2. **First Server Block (Static Files Server)**

```
server {
    listen 8080;
    root /Users/HusseinNasser/nginxcourse;
    ...
}

```

This defines a web server listening on port `8080`. The `root` directive specifies the directory from which Nginx will serve static files, in this case, `/Users/HusseinNasser/nginxcourse`.

### **Location for `/images` Directory**

```
location /images {
    root /Users/HusseinNasser/;
}

```

- This location block defines how Nginx will handle requests to `/images`. Instead of serving files from the root directory (`/Users/HusseinNasser/nginxcourse`), it will serve them from `/Users/HusseinNasser/`.
- For example, a request to `http://yourdomain.com/images/pic.jpg` will map to the physical path `/Users/HusseinNasser/images/pic.jpg`.

### **Location to Block `.jpg` Files**

```
location ~ .jpg$ {
    return 403;
}

```

- This location block uses a **regular expression** (`~ .jpg$`) to match any request that ends with `.jpg` (i.e., any JPG image file).
- Instead of serving the file, it returns a `403 Forbidden` error for any `.jpg` request. For example, requests like `http://yourdomain.com/pic.jpg` would be blocked with a `403 Forbidden` error.

---

### 3. **Second Server Block (Reverse Proxy Server)**

```
server {
    listen 8888;
    ...
}

```

This defines a second server that listens on port `8888`. This server acts as a **reverse proxy** that forwards requests to the first server (which is running on port `8080`).

### **Location `/`**

```
location / {
    proxy_pass http://localhost:8080/;
}

```

- Any requests to `http://yourdomain.com/` on port `8888` will be forwarded to `http://localhost:8080/` (the first server we defined above).
- This means the content from `/Users/HusseinNasser/nginxcourse` will be served when accessing the root URL (from port `8888`).

### **Location `/img`**

```
location /img {
    proxy_pass http://localhost:8080/images/;
}

```

- This forwards requests to `http://yourdomain.com/img` to the `/images/` directory on the first server running on port `8080`.
- A request like `http://yourdomain.com/img/someimage.jpg` will be forwarded to `http://localhost:8080/images/someimage.jpg` for file serving.

---

### 4. **Events Block**

```
events { }

```

As previously mentioned, this block is empty but required by Nginx. It manages worker processes and event handling.

---

### Summary of How it Works:

- **First Server (Port 8080)**:
    - Serves static files from `/Users/HusseinNasser/nginxcourse` by default.
    - Requests to `/images` are served from `/Users/HusseinNasser/` instead.
    - Any requests ending with `.jpg` will result in a `403 Forbidden` error.
- **Second Server (Port 8888)**:
    - Acts as a **reverse proxy** to the first server running on port `8080`.
    - For requests to `/`, it forwards them to the root (`/`) of the first server.
    - For requests to `/img`, it forwards them to the `/images` directory of the first server.

This configuration allows you to have two distinct Nginx servers:

1. A **static file server** (on port `8080`) for serving files directly.
2. A **reverse proxy** server (on port `8888`) that proxies requests to the first server with different rules.

[Back to Parent Page]({{ page.parent }})