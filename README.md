# Nginx

## 📌 1. Introduction

NGINX (pronounced "engine-x") is a high-performance web server, reverse proxy, load balancer, and caching system. It is widely used for handling concurrent connections efficiently due to its asynchronous, event-driven architecture.

### Why NGINX?

NGINX was originally designed to solve the C10K problem, which refers to handling 10,000 concurrent connections efficiently. Traditional web servers like Apache use a thread-based model, where each connection spawns a new thread, leading to high memory usage and CPU overhead. NGINX, however, is built around an event-driven model, allowing it to handle thousands of concurrent connections efficiently with a small memory footprint.

#### ✅ Key Advantages of NGINX:
- **Performance** – Handles more connections with fewer system resources.
- **Security** – Acts as a proxy to protect backend servers.
- **Load Balancing** – Distributes traffic among multiple servers.
- **Caching** – Stores responses for faster delivery.
- **Reverse Proxy** – Hides backend servers and forwards requests.

### 🚀 Key Features of NGINX
- **Reverse Proxy**: Acts as an intermediary between clients and backend servers.
- **Load Balancer**: Distributes incoming traffic across multiple backend servers.
- **Static Content Server**: Efficiently serves static files like HTML, CSS, and JavaScript.
- **Caching**: Stores frequently accessed data to reduce load on backend servers.
- **Security & Access Control**: Protects backend servers from direct exposure and attacks.
- **SSL/TLS Termination**: Encrypts connections for secure HTTPS traffic.
- **Compression & Streaming**: Optimizes bandwidth usage and improves page load speed.
- **Kubernetes Ingress Controller**: Manages external access to services in Kubernetes clusters.

### 💡 Example: When you visit Airbnb, NGINX acts as a reverse proxy between your browser and the backend servers.

---

## 📌 2. Installing and Running NGINX

### 2.1 Installing NGINX

#### 🖥 On macOS:
```nginx
brew install nginx
```

#### 🖥 On Linux (Ubuntu/Debian):
```nginx
sudo apt update
sudo apt install nginx
```

#### 🖥 On Windows:
Download the latest release from [NGINX Official Site](https://nginx.org/en/download.html).

### 2.2 Starting, Stopping, and Restarting NGINX
```nginx
sudo systemctl start nginx  # Start NGINX
sudo systemctl stop nginx   # Stop NGINX
sudo systemctl restart nginx  # Restart NGINX
sudo systemctl reload nginx  # Reload without downtime
```

### 2.3 Exploring NGINX Directory Structure

| Directory                     | Purpose                 |
|-------------------------------|-------------------------|
| `/etc/nginx/nginx.conf`       | Main configuration file |
| `/etc/nginx/sites-available/` | Available virtual hosts |
| `/etc/nginx/sites-enabled/`   | Enabled virtual hosts   |
| `/var/log/nginx/access.log`   | Access logs             |
| `/var/log/nginx/error.log`    | Error logs              |

---

## 📌 3. Understanding NGINX Configuration

### 3.1 NGINX Directives and Contexts

NGINX configuration consists of directives inside contexts. The main contexts are:
- **events** – Defines worker processes and connection handling.
- **http** – Contains settings for handling HTTP traffic.
- **server** – Defines server behavior, including listening ports.
- **location** – Specifies URL routing and request handling.

### Example NGINX Configuration:
```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include mime.types;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name localhost;

        location / {
            root /var/www/html;
            index index.html index.htm;
        }
    }
}
```

---

## 📌 4. Serving Static Content with NGINX

### Example Static File Server Configuration:
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        root /var/www/html;
        index index.html;
    }

    error_page 404 /404.html;
}
```

### Setting MIME Types for CSS, JS, and Images
```nginx
http {
    include mime.types;
    default_type application/octet-stream;
}
```

### Handling Custom 404 Pages
```nginx
server {
    error_page 404 /custom_404.html;
}

```
### Optimizing Static File Serving
Enable file compression and efficient file transfer:
```nginx
gzip on;
sendfile on;
```

---

## 📌 5. Reverse Proxy with NGINX

### Basic Reverse Proxy Setup
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend-server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 📌 6. Load Balancing with NGINX

### Round Robin Load Balancing
```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

---

## 📌 7. Secure Connections (HTTPS & SSL Termination)

```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt

```


### Enabling SSL
```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
}
```

### Redirecting HTTP to HTTPS

```nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}

```

---

## 📌 8. URL Routing, Rewrites, and Redirects

### Redirecting HTTP to HTTPS
```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### URL Rewrite
```nginx
server {
    rewrite ^/old/(.*)$ /new/$1 redirect;
}
```

---

## 📌 9. Caching and Performance Optimization

### Enable Proxy Caching
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m inactive=60m;
```

### Enable Gzip Compression
```nginx
gzip on;
gzip_types text/plain text/css application/json;
```

---

## 📌 10. Security and Access Control

### Block Unauthorized Access
```nginx
location /admin {
    deny all;
}
```

### Rate Limiting to Prevent DDoS
```nginx
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

---

## 📌 11. Using NGINX in Cloud & Kubernetes

### Deploying NGINX in Kubernetes

```nginx
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-service
            port:
              number: 80

```
---

## 📌 12. Security and Access Control

### Restricting Access

```nginx
location /admin/ {
    deny all;
    allow 192.168.1.1;
}

```

### Rate Limiting to Prevent DDoS

```nginx
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

```


---

## 📌 13. Debugging and Logging

### Enable Debugging Logs
```nginx
error_log /var/log/nginx/error.log debug;
```

### Check NGINX Configuration
```nginx
nginx -V
nginx -T
```

### Viewing Logs
```nginx
tail -f /var/log/nginx/access.log

```

## 📌 14. Monitoring and Logging with NGINX

- **Using NGINX Logs for Debugging**
- **Logging Client Requests**
- **Using `log_format` to Customize Logs**
- **Analyzing Performance with Access Logs**

#### Custom Log Format Example:
```nginx
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent"';

access_log /var/log/nginx/access.log main;
```



## 📌 15. Advanced Load Balancing Strategies

NGINX provides several **load balancing** methods to distribute traffic across backend servers efficiently. These strategies help optimize performance, reliability, and resource utilization.

### ⚖️ **Types of Load Balancing in NGINX**
1. **Round Robin (Default)** – Requests are distributed sequentially among backend servers.
2. **Least Connections** – Requests are sent to the server with the fewest active connections.
3. **IP Hash** – Ensures the same client IP is routed to the same backend server.
4. **Weighted Load Balancing** – Some servers receive more requests than others.
5. **Sticky Sessions (Session Persistence)** – Ensures requests from the same client always go to the same backend server.

---

### 🔄 **Round Robin Load Balancing (Default)**
_NGINX automatically distributes requests among backend servers in a round-robin fashion._

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
    }
}
```

### 📉 Least Connections Load Balancing
Requests are sent to the backend server with the fewest active connections.
```ngnix

upstream backend {
    least_conn;  # Enable Least Connections Load Balancing
    server backend1.example.com;
    server backend2.example.com;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```
### 🔗 IP Hash Load Balancing
Ensures that a client’s requests always go to the same backend server based on their IP.

```ngnix
upstream backend {
    ip_hash;  # Enable IP Hash Load Balancing
    server backend1.example.com;
    server backend2.example.com;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```

### ⚖️ Weighted Load Balancing
You can assign higher weights to more powerful servers so they handle more traffic.

``` ngnix

upstream backend {
    server backend1.example.com weight=3;  # This server gets more traffic
    server backend2.example.com weight=1;  # Less traffic
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```
### 🔄 Sticky Sessions (Session Persistence)
Ensures that a user always reaches the same backend server (useful for stateful applications).

```ngnix
upstream backend {
    ip_hash;  # Sticky session using client IP
    server backend1.example.com;
    server backend2.example.com;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```
### 📌 Load Balancing WebSockets & TCP Connections
NGINX can also balance WebSockets and TCP traffic efficiently.
```ngnix

upstream websocket_backend {
    server backend1.example.com:8080;
    server backend2.example.com:8080;
}

server {
    listen 80;

    location /ws/ {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
```







