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

📌 2. Installing and Running NGINX

  2.1 Installing NGINX <br> <br>
  🖥 On macOS: <br>
   `
   brew install nginx
   `
   <br> <br>
  🖥 On Linux (Ubuntu/Debian): <br>
  `sudo apt update <br>
   sudo apt install nginx
   `
    <br> <br>
🖥 On Windows: <br>
Download the latest release from NGINX Official Site.
 <br> <br>
2.2 Starting, Stopping, and Restarting NGINX <br>
  `
  sudo systemctl start nginx  <br>
  sudo systemctl stop nginx  <br>
  sudo systemctl restart nginx  <br>
  sudo systemctl reload nginx  # Reload without downtime
  `
 <br> <br>
