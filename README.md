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
```sh
brew install nginx
```

#### 🖥 On Linux (Ubuntu/Debian):
```sh
sudo apt update
sudo apt install nginx
```

#### 🖥 On Windows:
Download the latest release from [NGINX Official Site](https://nginx.org/en/download.html).

### 2.2 Starting, Stopping, and Restarting NGINX
```sh
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

### 3.1 Directives and Contexts

NGINX configuration consists of **directives** (key-value pairs) inside **contexts** (blocks of code).

#### Example: Basic NGINX Config
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

### Breakdown of the Configuration

#### 1. **Global Configuration**
```nginx
worker_processes auto;
```
This sets the number of worker processes. `auto` means NGINX will automatically set it to the number of CPU cores available on the system.

#### 2. **Events Block**
```nginx
events {
    worker_connections 1024;
}
```
`worker_connections` defines the maximum number of simultaneous connections each worker process can handle.

#### 3. **HTTP Block**
```nginx
http {
    include mime.types;
    sendfile on;
    keepalive_timeout 65;
```
- `include mime.types;` – Maps file extensions to MIME types.
- `sendfile on;` – Enables `sendfile()` system call for efficient file serving.
- `keepalive_timeout 65;` – Sets the timeout for keep-alive connections to 65 seconds.

#### 4. **Server Block**
```nginx
server {
    listen 80;
    server_name localhost;
```
- `listen 80;` – NGINX listens for HTTP requests on port 80.
- `server_name localhost;` – Specifies the domain name.

#### 5. **Location Block**
```nginx
    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}
```
- `location / {}` – Defines how NGINX handles requests to `/`.
- `root /var/www/html;` – Specifies the document root for serving files.
- `index index.html index.htm;` – Specifies default files to serve.

This basic configuration enables NGINX to serve static files efficiently and handle multiple connections simultaneously.

---

## 📌 4. Serving Static Content with NGINX

### 4.1 Configuring NGINX to Serve Static Files
```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    index index.html;
}
```

### 4.2 Setting MIME Types for CSS, JS, and Images
```nginx
http {
    include mime.types;
    default_type application/octet-stream;
}
```

### 4.3 Handling Custom 404 Pages
```nginx
server {
    error_page 404 /custom_404.html;
}
```

---

This document is a **comprehensive guide** to understanding and using NGINX efficiently. Continue exploring to learn about **reverse proxy, caching, load balancing, security, logging, and deploying NGINX in Kubernetes!** 🚀

