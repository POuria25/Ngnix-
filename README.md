# Ngnix


📌 1. Introduction

NGINX (pronounced "engine-x") is a high-performance web server, reverse proxy, load balancer, and caching system. It is widely used for handling concurrent connections efficiently due to its asynchronous, event-driven architecture.

  Why NGINX ?

  NGINX was originally designed to solve the C10K problem, which refers to handling 10,000 concurrent connections efficiently. Traditional web servers like Apache use a thread-based   model, where each connection spawns a new thread, leading to high memory usage and CPU overhead. NGINX, however, is built around an event-driven model, allowing it to handle         thousands of concurrent connections efficiently with a small memory footprint.

  
  ✅ Performance – Handles more connections with fewer system resources.
  
  ✅ Security – Acts as a proxy to protect backend servers.
  
  ✅ Load Balancing – Distributes traffic among multiple servers.
  
  ✅ Caching – Stores responses for faster delivery.
  
  ✅ Reverse Proxy – Hides backend servers and forwards requests.

  Key Features of NGINX 🚀 

  Reverse Proxy: Acts as an intermediary between clients and backend servers. 

  Load Balancer: Distributes incoming traffic across multiple backend servers.

  Static Content Server: Efficiently serves static files like HTML, CSS, and JavaScript.

  Caching: Stores frequently accessed data to reduce load on backend servers.

  Security & Access Control: Protects backend servers from direct exposure and attacks.

  SSL/TLS Termination: Encrypts connections for secure HTTPS traffic.

  Compression & Streaming: Optimizes bandwidth usage and improves page load speed.

  Kubernetes Ingress Controller: Manages external access to services in Kubernetes clusters.

  💡 Example: When you visit Airbnb, NGINX acts as a reverse proxy between your browser and the backend servers.

<br> <br> <br> 
📌 2. Installing and Running NGINX

  2.1 Installing NGINX <br> <br>
  🖥 On macOS: <br>
   `
   brew install nginx
   `
   <br> <br>
  🖥 On Linux (Ubuntu/Debian): <br>
  `sudo apt update ` <br>
  `sudo apt install nginx`  <br>
    <br> <br>
  🖥 On Windows: <br>
  Download the latest release from NGINX Official Site.
   <br> <br>
  2.2 Starting, Stopping, and Restarting NGINX <br>
  `sudo systemctl start nginx`  <br>
  `sudo systemctl stop nginx`  <br>
  `sudo systemctl restart nginx`  <br>
  `sudo systemctl reload nginx  # Reload without downtime` 
 <br> <br>

  2.3 Exploring NGINX Directory Structure
  <br> <br>
  | Directory                     | Purpose                 |
  |-------------------------------|-------------------------|
  | `/etc/nginx/nginx.conf`       | Main configuration file |
  | `/etc/nginx/sites-available/` | Available virtual hosts |
  | `/etc/nginx/sites-enabled/	` | Enabled virtual hosts   |
  | `/var/log/nginx/access.log	` | Access logs             |
  | `/var/log/nginx/error.log	 `  | Error logs              |

  <br><br><br>

  
  📌 3. Understanding NGINX Configuration <br><br>
  3.1 Directives and Contexts<br>
  NGINX configuration consists of directives (key-value pairs) inside contexts (blocks of code).<br><br>
  Example: Basic NGINX Config


 ```
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
<br>
1. Global Configuration

```
worker_processes auto;
```
This sets the number of worker processes. auto means Nginx will automatically set it to the number of CPU cores available on the system.

```
events {
    worker_connections 1024;
}
```
worker_connections defines the maximum number of simultaneous connections each worker process can handle. In this case, each worker can handle up to 1024 connections.


2. HTTP Block

```
http {
    include mime.types;
    sendfile on;
    keepalive_timeout 65;
```

include mime.types; 👉🏻 includes the MIME types file, which maps file extensions to MIME types (e.g., .html to text/html).

sendfile on; 👉🏻 enables the use of sendfile() system call, which is more efficient for serving static files.

keepalive_timeout 65; 👉🏻 sets the timeout for keep-alive connections to 65 seconds.
