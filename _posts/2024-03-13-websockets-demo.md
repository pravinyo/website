---
title: "NGINX: Websockets Demo"
author: pravin_tripathi
date: 2024-03-13 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/websockets-demo/
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

Node.js server code

```jsx
const http = require("http");
const WebSocketServer = require("websocket").server
const PORT = process.argv[2] || 8080;
let connection = null;

//create a raw http server (this will help us create the TCP which will then pass to the websocket to do the job)
const httpserver = http.createServer();

 //pass the httpserver object to the WebSocketServer library to do all the job, this class will override the req/res
const websocket = new WebSocketServer({
    "httpServer": httpserver
})

httpserver.listen(PORT, () => console.log(`My server is listening on port ${PORT}`))

//when a legit websocket request comes listen to it and get the connection .. once you get a connection thats it!
websocket.on("request", request=> {

    connection = request.accept(null, request.origin)
    connection.on("close", () => console.log("CLOSED!!!"))
    connection.on("message", message => {

        console.log(`Received message ${message.utf8Data}`)
        connection.send(`Server ${PORT} responded to your message: ${message.utf8Data}`)
    })

})
```

## Configure Nginx as Layer 4 WebSocket Proxy

```config
stream {
    
    upstream allbackend {
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
        server 127.0.0.1:5555;
    }
    
    server {
          listen 8080;
          proxy_pass allbackend;
 
     }
}

events { } 
```

- Listening on port 8080
- Any TCP connection request is a tunnel and always goes to the websocket app
- Paths don’t matter (layer 7)
    - ws[://localhost/](http://localhost/wsapp) -> websocket app
    - ws://localhost/blahblah -> websocket app
- Layer 4 proxying blindly tunnels everything to the backend
- Any connection request to port 8080 will be tunneled to the websocket app backend

### Run below code to connect to webSockets

```jsx

let ws = new WebSocket("ws://localhost:8080");
ws.onmessage = message => console.log(`Received: ${message.data}`);
ws.send("Hello! I'm client")
```

`ws` create a tunnel which is sticky and request goes to same server. If any path is used here, it will not be checked and request is blindly passed to server.

## Configure Nginx as Layer 7 WebSocket Proxy

- Intercept the path and “route” appropriately
- [http://localhost/](http://localhost/) -> open main html page
- ws -> websocket app
    
    ws[://localhost/wsapp](http://localhost/wsapp)
    
- ws/chat -> another websocket app for chatting
    
    ws[://localhost](http://localhost/wsapp)/chat
    
- Can’t do that in Layer 4 since port 80 is blindly tunnels

### HTML frontend code:

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <h1>Hello, WebSockets</h1>
    <input type = 'text' id = 'txt'>
    <div id = 'divOut'></div>
    <script>
    const txtBox = document.getElementById("txt");
    const divOut = document.getElementById("divOut");
    const ws = new WebSocket("ws://localhost/wsapp/")
    ws.onmessage = e=> divOut.innerHTML += e.data + "<br/>";
    txtBox.addEventListener("keypress", e=> {
        if (e.keyCode === 13){

          ws.send(txtBox.value);
        txtBox.value ="";
        }
    })
    </script>

</body>
</html>
```

### Nginx config

```config
http {

    upstream allbackend {
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
        server 127.0.0.1:5555;
    }

    server {
        listen 80;
        location / {
             root /Users/HusseinNasser/javascript/javascript_playground/nginx-websockets/;
           }

      
				location /wsapp/ {
				    proxy_pass http://allbackend;
				    proxy_http_version 1.1;
				    proxy_set_header Upgrade $http_upgrade;
				    proxy_set_header Connection "Upgrade";
				    proxy_set_header Host $host;
				}
    }
}

# due to some reasons, NGINX requires this block!
events { }
```

[Back to Parent Page]({{ page.parent }})