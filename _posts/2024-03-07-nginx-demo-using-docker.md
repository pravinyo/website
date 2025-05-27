---
title: "NGINX: NGINX demo using Docker"
author: pravin_tripathi
date: 2024-03-07 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
attachment_path: /assets/document/attachment/nginx-demo-using-docker/
permalink: /nginx-understanding-and-deployment/nginx-demo-using-docker/
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

# Nginx demo using Docker

## Docker command to install and run nginx

docker run --name nginx --hostname ng1 -p 80:80 -d nginx

//inspect ccontainer
docker inspect nginx

### index.js

```jsx
const express = require("express");
const app = express();
const os = require("os")
const hostname = os.hostname();

app.get("/", (req, res) => res.send("Hello from " + hostname ) ) ;
app.listen(8080, () => console.log("listening on port 8080 on " + hostname));
```

### package.json

```jsx
{
  "name": "app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2"
  }
}

```

### Install and run app

```jsx

npm install
node index.js
```

### Dockerfile

```docker
FROM node:12
WORKDIR /home/node/app
COPY app /home/node/app
RUN npm install
CMD node index.js
```

### create  nodeapp

docker build . -t nodeapp
docker run --hostname nodeapp1 --name nodeapp1 -d nodeapp
docker run --hostname nodeapp2 --name nodeapp2 -d nodeapp
docker run --hostname nodeapp3 --name nodeapp3 -d nodeapp

### nginx.config

```config
http {
    upstream nodebackend {
         server nodeapp1:8080;
				 server nodeapp2:8080;
				 server nodeapp3:8080;
    }

   server {
        listen 8080;
        location / {
             proxy_pass http://nodebackend/;
         }
    }

}

events
{

}
```

### Start Nginx

```docker
docker run --hostname ng1 --name nginx -p 80:8080 -v /Users/pravin.tripathi/Desktop/nginx-udemy-container/nginx.conf:/etc/nginx/nginx.conf -d nginx
```

you will find something like below

```docker
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/11/21 09:57:08 [emerg] 1#1: host not found in upstream "nodeapp1:8080" in /etc/nginx/nginx.conf:6
nginx: [emerg] host not found in upstream "nodeapp1:8080" in /etc/nginx/nginx.conf:6
```

In reality nginx stopped . all are running just that they are not able to interact with each other and nginx throw error and goes down. We have to put all of them in same network.

```docker
docker network create backendnet
docker network connect backendnet nodeapp1
docker network connect backendnet nodeapp2
docker network connect backendnet nodeapp3
docker network connect backendnet nginx
```

```docker
docker start nginx
```

```docker
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/11/21 09:57:08 [emerg] 1#1: host not found in upstream "nodeapp1:8080" in /etc/nginx/nginx.conf:6
nginx: [emerg] host not found in upstream "nodeapp1:8080" in /etc/nginx/nginx.conf:6
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: IPv6 listen already enabled
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```

It is up now.

![image.png](nginx-demo-using-docker/image.png)

Let spin up another nginx container in above network.

```docker
docker run --hostname ng2 --name nginx2 -p 81:8080 -v /Users/pravin.tripathi/Desktop/nginx-udemy-container/nginx.conf:/etc/nginx/nginx.conf -d nginx
```

Add to network,

```docker
docker network connect backendnet nginx2
```

[nginx-udemy-container 2.zip]({{page.attachment_path}}/nginx-udemy-container_2.zip)

[Back to Parent Page]({{ page.parent }})