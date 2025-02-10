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

### Enabling SSL
```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
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
This README now includes detailed explanations and relevant code snippets for each section. 🚀

