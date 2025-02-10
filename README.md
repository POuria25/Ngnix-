# Nginx

## 👉 1. Introduction

NGINX (pronounced "engine-x") is a high-performance web server, reverse proxy, load balancer, and caching system. It is widely used for handling concurrent connections efficiently due to its asynchronous, event-driven architecture.

### Why NGINX?

NGINX was originally designed to solve the C10K problem, which refers to handling 10,000 concurrent connections efficiently. Traditional web servers like Apache use a thread-based model, where each connection spawns a new thread, leading to high memory usage and CPU overhead. NGINX, however, is built around an event-driven model, allowing it to handle thousands of concurrent connections efficiently with a small memory footprint.

**Key Advantages:**

- 💡 **Performance** – Handles more connections with fewer system resources.
- 💪 **Security** – Acts as a proxy to protect backend servers.
- 🎯 **Load Balancing** – Distributes traffic among multiple servers.
- ⏳ **Caching** – Stores responses for faster delivery.
- 👀 **Reverse Proxy** – Hides backend servers and forwards requests.

### Key Features of NGINX 🚀

- **Reverse Proxy**: Acts as an intermediary between clients and backend servers.
- **Load Balancer**: Distributes incoming traffic across multiple backend servers.
- **Static Content Server**: Efficiently serves static files like HTML, CSS, and JavaScript.
- **Caching**: Stores frequently accessed data to reduce load on backend servers.
- **Security & Access Control**: Protects backend servers from direct exposure and attacks.
- **SSL/TLS Termination**: Encrypts connections for secure HTTPS traffic.
- **Compression & Streaming**: Optimizes bandwidth usage and improves page load speed.
- **Kubernetes Ingress Controller**: Manages external access to services in Kubernetes clusters.

💡 **Example**: When you visit Airbnb, NGINX acts as a reverse proxy between your browser and the backend servers.

---

## 👉 2. Installing and Running NGINX

### 2.1 Installing NGINX

**On macOS:**
```sh
brew install nginx
```

**On Linux (Ubuntu/Debian):**
```sh
sudo apt update
sudo apt install nginx
```

**On Windows:**
Download the latest release from [NGINX Official Site](https://nginx.org/en/download.html).

### 2.2 Starting, Stopping, and Restarting NGINX
```sh
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
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

## 👉 3. Understanding NGINX Configuration

### 3.1 Directives and Contexts
NGINX configuration consists of **directives** (key-value pairs) inside **contexts** (blocks of code).

#### Example: Basic NGINX Config

```nginx
worker_processes auto;   # Number of worker processes, auto means CPU core count

events {
    worker_connections 1024;  # Max connections per worker
}

http {
    include mime.types;  # MIME type mapping
    sendfile on;  # Optimize file serving
    keepalive_timeout 65;

    server {
        listen 80;  # Listen on port 80
        server_name localhost;

        location / {
            root /var/www/html;
            index index.html index.htm;
        }
    }
}
```

### Breakdown of Key Sections:

#### 1. **Global Configuration**
```nginx
worker_processes auto;
```
- Sets the number of worker processes. `auto` means Nginx will automatically set it to the number of CPU cores available on the system.

```nginx
events {
    worker_connections 1024;
}
```
- `worker_connections` defines the maximum number of simultaneous connections each worker process can handle.

#### 2. **HTTP Block**
```nginx
http {
    include mime.types;
    sendfile on;
    keepalive_timeout 65;
```
- `include mime.types;` 💡 Includes the MIME types file, mapping file extensions to MIME types (e.g., .html to text/html).
- `sendfile on;` 💡 Enables the `sendfile()` system call for efficient file serving.
- `keepalive_timeout 65;` 💡 Sets the timeout for keep-alive connections to 65 seconds.

#### 3. **Server Block**
```nginx
server {
    listen 80;
    server_name localhost;
```
- `listen 80;` 💡 Tells NGINX to listen on port 80 for incoming HTTP requests.
- `server_name localhost;` 💡 Specifies that this server block will handle requests for localhost.

```nginx
    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}
```
- `location / {}` 💡 Defines how NGINX should handle requests to the root URL (`/`).
- `root /var/www/html;` 💡 Sets the root directory for serving files.
- `index index.html index.htm;` 💡 Specifies the default files to serve.

### Summary:
- This configuration sets up NGINX to serve static files from `/var/www/html` on port 80.
- Uses the number of CPU cores for worker processes and allows each worker to handle up to 1024 connections.
- The `sendfile` directive is enabled for efficient file serving.
- Keep-alive connections are set to timeout after 65 seconds.

---

## 👉 More Advanced Topics (Coming Soon)

- 💡 **Serving Static Content**
- 💡 **Reverse Proxy & Load Balancing**
- 💡 **SSL/TLS Encryption (HTTPS Setup)**
- 💡 **Security Best Practices**
- 💡 **NGINX with Kubernetes (Ingress Controller)**
- 💡 **Caching & Performance Optimization**
- 💡 **Monitoring & Debugging NGINX Logs**

Stay tuned for more updates! 🚀🔥

