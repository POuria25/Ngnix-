# Nginx&#32;

## 1. What nginx is

**Define.** Nginx is an open-source web server that also works as a reverse proxy, load balancer, HTTP cache, TLS terminator and API gateway. It is used by Netflix, GitHub, Cisco, LinkedIn and others; the `Server` response header in the browser's Network tab is a clue to what a site runs, not proof, since it can be removed or changed.

**Problem.** An application server (Node, Laravel, ASP.NET) that handles both business logic and thousands of raw connections hits a bottleneck. One server is also a single point of failure and a large attack surface if every instance is public.

**Mechanism.** Clients talk only to nginx; nginx decides what to do with each request.

| Role | What nginx does | Benefit |
| --- | --- | --- |
| Web server | Returns static files (HTML, CSS, JS, images) directly | Fast, low CPU and memory |
| Reverse proxy | Accepts the request on behalf of backends and forwards it | Backends hidden; one public entry point |
| Load balancer | Spreads requests across several backends (round robin, least\_conn, weights, backup) | Scale, availability |
| TLS terminator | Handles HTTPS in one place, forwards plain HTTP inside | One certificate setup instead of N |
| Cache | Stores backend responses and replays them | Fewer backend and DB hits |
| Compressor | gzip (or brotli) before sending | Less bandwidth |
| Security layer | Hides versions, adds headers, rate limits, rejects bad hosts | Smaller attack surface |

A forward proxy sits on the client side (hides clients, controls their internet access). A reverse proxy sits on the server side (hides servers). Nginx here is the second kind. Nginx also streams responses in chunks (`Transfer-Encoding: chunked`, RFC 9112), so a client can start receiving data before the response is complete.

**Limitations.**

- A single nginx is itself a single point of failure; production needs a second node or a cloud load balancer in front.
- Claim to flag: "nginx is faster than Apache." Nginx's event-driven design uses less memory per connection, but Apache's event MPM narrows the gap; the sources present it as a design trade-off (Apache: flexibility, `.htaccess`, module ecosystem), not a replacement.
- Inside Kubernetes, the NGINX Ingress controller plays the same proxy role but is not publicly exposed by itself; a cloud load balancer (for example AWS ELB) is the public entry and forwards to it.

**Cross-links.** HTTP semantics: RFC 9110. Caching: RFC 9111. TLS 1.3: RFC 8446. Gzip: RFC 1952. Brotli: RFC 7932. Forwarded-client info: RFC 7239. Certificates via ACME: RFC 8555.

## 2. Install, file layout, config anatomy

### Install

Ubuntu/Debian VPS (connect with `ssh user@SERVER_IP`):

```bash
sudo apt update
sudo apt install nginx
nginx -v                       # version
sudo systemctl status nginx    # active (running); Ctrl+C to leave
# browse to http://SERVER_IP -> "Welcome to nginx"
```

macOS (Homebrew):

```bash
brew install nginx
nginx -v
whereis nginx
nginx -V                       # build info, incl. config file path
nginx                          # start
nginx -s reload                # reload config
nginx -s quit                  # stop gracefully
nginx -h                       # list subcommands
# default test page: http://localhost:8080
```

Windows: use the official zip; the Laith Harb tutorial links to a guide. Find the active config path on any OS with `nginx -V` and look for `--conf-path`. Typical locations: `/etc/nginx/nginx.conf`, `/usr/local/etc/nginx/nginx.conf` (Homebrew), `/usr/local/nginx/conf`.

Opening the config in VS Code from the terminal: Command Palette (Cmd+Shift+P) then "Shell Command: Install 'code' command in PATH", then `code /path/to/nginx.conf`.

### Directory layout (Debian/Ubuntu package)

| Path | Purpose |
| --- | --- |
| `/etc/nginx/nginx.conf` | Main file: global settings, `events`, `http`, and the `include` lines |
| `/etc/nginx/sites-available/` | One file per site/app (your server blocks live here) |
| `/etc/nginx/sites-enabled/` | Symlinks to the sites that are active; nginx loads only these |
| `/etc/nginx/conf.d/` | Extra `*.conf` files; the official Docker image uses this instead of sites-\* |
| `/etc/nginx/snippets/` | Reusable fragments (e.g. common SSL settings) |
| `/etc/nginx/mime.types` | File-extension to content-type map |
| `/var/www/html` | Default web root (holds the welcome page) |

### Directives and contexts

A **directive** is a `key value;` pair and must end with a semicolon. A **context** is a `{ ... }` block that groups directives. The minimal skeleton:

```nginx
worker_processes auto;          # number of worker processes; "auto" = one per CPU core
user www-data;                  # (Debian) user that runs workers

events {
    worker_connections 1024;    # simultaneous connections per worker
}

http {
    include mime.types;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;

    server {
        listen 8080;
        server_name localhost;
        # location blocks go here
    }
}
```

- `worker_processes 1;` was used for the demos. In production set it to the CPU core count, or `auto`.
- `events` is mandatory even if empty. `worker_processes x worker_connections` counts connection slots, not clients: a reverse proxy uses two slots per request and every slot is a file descriptor (section 10); higher values cost memory.
- You can put a `server` block directly in `nginx.conf`, but the recommended practice is one file per site in `sites-available` plus a symlink.
- The tutorials start the nginx.conf from scratch (delete the boilerplate); on a package install keep the `include` lines.

### Test, reload, restart

```bash
sudo nginx -t                    # syntax check; always run before reloading
sudo systemctl reload nginx      # apply config without dropping connections (preferred)
sudo systemctl restart nginx     # full restart
sudo systemctl stop nginx
sudo systemctl disable nginx     # do not start at boot
sudo systemctl enable nginx
```

Common failure shown in the VPS tutorial: a missing `;` makes `nginx -t` fail; fix it and re-run. A symlink only needs to be created once; later edits to the file in `sites-available` take effect on the next reload.

## 3. Serving static content

### First site (package install, sites-available pattern)

```bash
sudo mkdir /var/www/my-first-site
sudo nano /var/www/my-first-site/index.html
```

```html
<h1>Hello from Sarvin style coding</h1>
```

```bash
sudo nano /etc/nginx/sites-available/first-site
```

```nginx
server {
    listen 90;                       # port this server answers on
    server_name localhost;           # which Host header it answers to

    location / {                     # "/" matches every path
        root /var/www/my-first-site; # where files live
        index index.html;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/first-site /etc/nginx/sites-enabled/first-site
sudo nginx -t
sudo systemctl reload nginx
# http://SERVER_IP:90
```

Use `reload`, not `restart`: existing connections are not interrupted.

### Same idea locally (Laith Harb, macOS)

```bash
mkdir ~/Desktop/mysite && cd ~/Desktop/mysite && code .
```

`index.html`:

```html
<html><body><h1>Hello my friends, I am served from nginx</h1></body></html>
```

Full `nginx.conf`:

```nginx
events {}

http {
    server {
        listen 8080;
        root /Users/<you>/Desktop/mysite;   # nginx serves index.html from here
    }
}
```

```bash
nginx -s reload      # then open http://localhost:8080
```

### MIME types

Without types, a CSS file is sent as `text/plain` and the browser ignores it. Add `styles.css` (`h1 { background-color: pink; color: aqua; }`) and link it from the page head:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>My nginx project</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body><h1>Hello my friends, I am served from nginx</h1></body>
</html>
```

Manual fix (works but does not scale):

```nginx
http {
    types {
        text/html  html;
        text/css   css;
    }
    server { listen 8080; root /Users/<you>/Desktop/mysite; }
}
```

Better: include the shipped map (it already covers gif, jpeg, js, ...):

```nginx
http {
    include mime.types;
    server { listen 8080; root /Users/<you>/Desktop/mysite; }
}
```

If the style still does not apply, do a hard reload in the browser (Cmd+Shift+R); it was cached as `text/plain`. Also useful: the `include mime.types;` line is what puts the correct `Content-Type` in responses for HTML, images, JS and CSS.

### location, root vs alias, try\_files, regex

Create `mysite/fruits/index.html` containing a `<ul>` with Mango, Strawberry, Watermelon.

```nginx
http {
    include mime.types;

    server {
        listen 8080;
        root /Users/<you>/Desktop/mysite;

        location /fruits {
            root /Users/<you>/Desktop/mysite;
            # request /fruits -> file mysite/fruits/index.html
            # root APPENDS the location path to the root
        }

        location /carbs {
            alias /Users/<you>/Desktop/mysite/fruits;
            # alias REPLACES the location path; no "carbs" dir needed
        }

        location /vegetables {
            root /Users/<you>/Desktop/mysite;
            try_files /vegetables/veggies.html /index.html =404;
            # try in order; =404 if none exists
        }

        location ~* /count/[0-9] {          # ~* = case-insensitive regex
            root /Users/<you>/Desktop/mysite;
            try_files /index.html =404;
        }
    }
}
```

- `root` builds the file path as root + request URI; `alias` replaces the matched location prefix (details in section 11). A directory without `index.html` gives 403, not 404 (observed with `/vegetables`).
- `try_files` checks candidates left to right; the last item is the fallback (a file, a URI, or a status like `=404`).
- `/count/4`, `/count/5`, `/count/6` all return the root `index.html`.
- Location prefixes: no modifier = prefix match, `=` exact, `~` regex (case-sensitive), `~*` regex (case-insensitive).

### Redirects vs rewrites

```nginx
# Redirect: browser URL changes to /fruits
location /crops {
    return 307 /fruits;
}

# Rewrite: URL in the browser stays /number/3, nginx internally serves /count/3
rewrite ^/number/(\w+) /count/$1;
```

Rule of thumb: `return 3xx` tells the client to go elsewhere; `rewrite` changes the URI nginx processes internally, so the address bar does not change. 301 is permanent, 307 keeps the HTTP method.

## 4. Reverse proxy

### The demo applications

Three sources, three backends; all follow the same pattern.

- **Full-stack project (Sarvin):** React front end on 3000, Node/Express API on 3001, Mongo Express on 8081, plus MongoDB. Without nginx each service needs its own port; with nginx all go through port 80.
- **Node static app (TechWorld):** one `server.js` (Express) serving `index.html` and `/images` on port 3000, run as three replicas on host ports 3001 to 3003.
- **Tiny Express API (Laith):** one `GET /` returning "I am an endpoint", run as four containers on 1111 to 4444.

TechWorld's `server.js` (served files plus a log line that names the replica):

```javascript
const express = require('express');
const path = require('path');
const app = express();

app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'index.html'));
  console.log('request served by ' + process.env.APP_NAME);
});
app.use('/images', express.static(path.join(__dirname, 'images')));

app.listen(3000, () => console.log('app listening on port 3000'));
```

`package.json` dependencies were `express` (and a second small package); install with `npm install`, run with `node server.js`.

Laith's API:

```javascript
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('I am an endpoint'));
app.listen(7777, () => console.log('listening on port 7777'));
```

### Path-based proxy to several services (VPS, nginx on the host)

```nginx
server {
    listen 80;
    server_name localhost;          # later: your domain

    location / {                    # React front end
        proxy_pass http://localhost:3000;
    }
    location /api/ {                # Node API: note the trailing slash
        proxy_pass http://localhost:3001/;
    }
    location /mongo/ {              # Mongo Express
        proxy_pass http://localhost:8081;
    }
}
```

Enable it and remove the default site (otherwise you still see the welcome page):

```bash
sudo ln -s /etc/nginx/sites-available/docker-demo /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default      # only removes the link; the file stays in sites-available
sudo nginx -t && sudo systemctl reload nginx
```

### Pitfall 1: trailing slash in proxy\_pass

| Config | Request | URI the backend receives |
| --- | --- | --- |
| `location /api/ { proxy_pass http://localhost:3001; }` | `/api/users` | `/api/users` (full URI forwarded) |
| `location /api/ { proxy_pass http://localhost:3001/; }` | `/api/users` | `/users` (matched prefix replaced) |

In the demo the Node app exposes `/users`, so the slash fixed the 404.

### Pitfall 2: apps that assume they live at "/"

Mongo Express returned "Cannot GET /" under `/mongo/`. Fix in `docker-compose.yml`, so it generates links under the sub-path:

```yaml
  mongo-express:
    environment:
      - ME_CONFIG_SITE_BASEURL=/mongo/
```

Then `git pull`, `docker compose down`, `docker compose up -d`.

### Domain names

At the DNS provider (Cloudflare in the demo) create A records for `example.com` and `www.example.com` pointing to the VPS IP, then set the same names in nginx and reload:

```nginx
server_name example.com www.example.com;
```

### Proxy headers (TechWorld)

The backend never sees the client directly, so forward what it needs:

```nginx
upstream nodejs_cluster {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 8080;
    server_name localhost;

    location / {
        proxy_pass http://nodejs_cluster;
        proxy_set_header Host $host;                  # original host the client asked for
        proxy_set_header X-Real-IP $remote_addr;      # real client IP, not nginx's
    }
}
```

Backends use these for routing, logging, analytics and security decisions. The standardized equivalent is the `Forwarded` header (RFC 7239); `X-Forwarded-For` and `X-Forwarded-Proto` are the de-facto variants you will also see.

Test the 5xx path: stop the backends and request the proxy; nginx answers 502 Bad Gateway with its own error page (its version string appears in it).

Limitation worth flagging: on a real server only nginx's port should be reachable from outside. In the local demo the container ports 3001 to 3003 are also open, which only works because it is localhost; use a firewall or leave the `ports:` mapping out of the backends in production.

## 5. Docker and Compose deployment

The transcripts describe the files aloud; the snippets below are reconstructed from that narration, so check ports and paths against your own project.

### Dockerfile for a Node app

TechWorld's version copies the files explicitly; Laith's and Sarvin's use the standard `package.json`-first pattern for layer caching.

```dockerfile
# Node + npm, same as installed locally
FROM node:20-alpine
# app folder inside the container
WORKDIR /app
# copy manifest first so the npm install layer is cached
COPY package.json .
RUN npm install
COPY server.js .
COPY index.html .
# index.html expects /app/images
COPY images ./images
# documentation only; publishing happens via compose ports or docker run -p
EXPOSE 3000
CMD ["node", "server.js"]
```

Build and run one container by hand:

```bash
docker build -t my-app:1.0 .
docker images
docker run -p 3000:3000 my-app:1.0     # host:container
```

Laith's four-instance variant (container always on 7777, different host ports):

```bash
docker build -t my-server .
docker run -d -p 1111:7777 my-server
docker run -d -p 2222:7777 my-server
docker run -d -p 3333:7777 my-server
docker run -d -p 4444:7777 my-server
```

### Compose with three replicas (TechWorld)

```yaml
services:
  app1:
    build: .
    environment:
      - APP_NAME=app1
    ports:
      - "3001:3000"
  app2:
    build: .
    environment:
      - APP_NAME=app2
    ports:
      - "3002:3000"
  app3:
    build: .
    environment:
      - APP_NAME=app3
    ports:
      - "3003:3000"
```

```bash
docker compose up --build -d
docker ps
docker compose logs -f
docker compose down
```

`APP_NAME` is read in Node as `process.env.APP_NAME`, so each response or log line shows which replica handled it. The Sarvin course does the same with `API_NAME` for `api`, `api2`, `api3` (copy the `api` service twice).

### Moving nginx itself into Compose (Sarvin)

First free port 80 and the old stack:

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx     # do not come back after reboot
cd ~/apps/docker-test-app && docker compose down
```

Put the server block in the repo as `nginx/default.conf` and add a service:

```yaml
  nginx:
    image: nginx:alpine           # pick a small alpine tag from Docker Hub
    ports:
      - "80:80"
      - "443:443"                  # added later for HTTPS
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - web
      - api
    restart: unless-stopped
```

Important changes when you containerise:

- The official image loads `/etc/nginx/conf.d/*.conf`; it does not use `sites-available` / `sites-enabled`, which are Debian/Ubuntu packaging conventions.
- `localhost` inside the nginx container means the nginx container itself. Replace it with Compose service names:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://web:3000;
    }
    location /api/ {
        proxy_pass http://api:3001/;
    }
    location /mongo/ {
        proxy_pass http://express:8081;
    }
}
```

- The front end's API URL must point to `/api` (relative, through nginx), not a host port.
- `restart: unless-stopped` restarts after a reboot but not after a manual `docker stop`.

Deploy and verify:

```bash
git pull
docker compose down
docker compose up --build -d
docker ps
docker exec -it <nginx-container-name> nginx -t     # validate config inside the container
docker exec <nginx-container-name> nginx -s reload  # reload without restart
```

Switching back to host nginx is the reverse: `docker compose down` the nginx stack, `sudo systemctl enable nginx && sudo systemctl start nginx`, then bring the app stack up.

Benefit: the nginx config is version-controlled with the code and the whole stack moves to another server with `git clone` and `docker compose up`.

Deployment note: the `docker compose down` then `up` sequence used above stops every container and drops connections. For updates without downtime use the workflow in section 18 (validate, update one service, reload, smoke-test, roll back on failure).

## 6. HTTPS

**Define.** HTTPS is HTTP over TLS (RFC 8446 for TLS 1.3, RFC 9110 for HTTP semantics). The server presents a certificate containing its public key; the private key stays on the server. A certificate authority (CA) signs the certificate so browsers trust it.

**Problem.** Plain HTTP exposes passwords and payment data to anyone on the path. Configuring TLS on every backend multiplies the work.

**Mechanism.** SSL/TLS termination at nginx: the client opens a secure connection to nginx, nginx decrypts and forwards to the backends. Port 443 is HTTPS, port 80 is HTTP.

Contested premise: the TechWorld tutorial notes that passing the encrypted traffic through to the backends so they decrypt it themselves is "a better security practice"; the Sarvin course and most small deployments terminate at nginx for simplicity. Both exist (terminate vs TLS passthrough or re-encryption); the right choice depends on whether the network between nginx and the backends is trusted.

### Option A: self-signed certificate (local testing)

The command was described but not shown in full; this is the standard form that matches the narration (RSA 2048, 365 days, key and certificate output paths):

```bash
mkdir ~/nginx-certs && cd ~/nginx-certs
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout nginx-selfsigned.key -out nginx-selfsigned.crt
# prompts: country (2 letters), state, city, organization, common name (localhost) ...
ls     # shows the .key (private) and .crt (certificate with public key)
```

Browsers warn on self-signed certificates because no CA vouches for them; click Advanced, then proceed to localhost. Never use this in production.

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /Users/<you>/nginx-certs/nginx-selfsigned.crt;   # public
    ssl_certificate_key /Users/<you>/nginx-certs/nginx-selfsigned.key;   # private

    location / {
        proxy_pass http://nodejs_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# redirect plain HTTP to HTTPS
server {
    listen 80;                   # TechWorld first used 8080, then switched to 80
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

```bash
nginx -s reload
# https://localhost  (443 is the default, no port in the URL)
```

### Option B: Let's Encrypt with Certbot, nginx on the host (Sarvin)

Prerequisite: the domain's A records already point to the server and port 80 is reachable (ACME HTTP-01 challenge, RFC 8555).

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx
```

Prompts: e-mail (renewal notices), accept the terms (`A`), EFF mailing list (optional, `N`), select domains (`1,2` for apex and www, comma or space separated), and redirect HTTP to HTTPS (choose `2`).

Certbot rewrites your site file. Typical result:

```nginx
server {
    server_name example.com www.example.com;
    # ... your locations ...
    listen 443 ssl;   # managed by Certbot
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I https://example.com
```

### Option C: HTTPS with nginx in Docker (Sarvin)

The certificates already exist on the host from Option B; mount them read-only into the container.

```yaml
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - /etc/letsencrypt:/etc/letsencrypt:ro     # host path : container path : read-only
    restart: unless-stopped
```

`nginx/default.conf` (HTTP redirects, HTTPS proxies):

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;     # $host = domain, $request_uri = original path
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / { proxy_pass http://web:3000; }
    location /api/ { proxy_pass http://api:3001/; }
    location /mongo/ { proxy_pass http://express:8081; }
}
```

```bash
docker compose down && docker compose up --build -d
docker exec -it <nginx-container> nginx -t
```

**Limitations.** Self-signed certificates give encryption without identity. Let's Encrypt certificates last 90 days, so renewal must keep working: once nginx runs in a container and holds port 80, the host's `certbot --nginx` plugin no longer applies; the sources do not cover renewal in Docker (common fixes: a certbot container with the webroot method, or `certbot renew` with a hook that reloads the container). Test the renewal path before relying on this setup.

**Cross-links.** Sections 4 (proxy), 9 (security headers), RFC 8555 (ACME), RFC 8446 (TLS 1.3).

## 7. Load balancing

**Define.** Nginx distributes incoming requests across several identical backends declared in an `upstream` block.

**Problem.** One backend has finite CPU and memory, and if it dies the application is down. Several instances (on one VPS in the demos, on separate servers in production) fix capacity and availability.

**Mechanism.** `upstream` names the group; `proxy_pass` points at the group name; the algorithm decides the target.

Host-level nginx, backends on localhost ports (TechWorld and Laith):

```nginx
http {
    include mime.types;

    upstream backendserver {                 # any name; must match proxy_pass
        server 127.0.0.1:1111;
        server 127.0.0.1:2222;
        server 127.0.0.1:3333;
        server 127.0.0.1:4444;
    }

    server {
        listen 8080;
        location / {
            proxy_pass http://backendserver;
        }
    }
}
```

Nginx in Docker, backends reached by Compose service name (Sarvin; same Docker network, so no IPs):

```nginx
upstream backend_servers {
    server api:3001;
    server api2:3001;
    server api3:3001;
}

server {
    listen 443 ssl;
    # ... ssl_certificate lines ...
    location /api/ {
        proxy_pass http://backend_servers/;   # was http://api:3001/
    }
}
```

Default algorithm is round robin: request 1 to api, 2 to api2, 3 to api3, then back to api. Verify by refreshing the page with DevTools Network open (responses carry the `API_NAME`), or watch `docker compose logs -f`.

### Other methods

```nginx
# Least connections: send to the server with the fewest active connections
upstream backend_servers {
    least_conn;
    server api:3001;
    server api2:3001;
    server api3:3001;
}

# Weights: api gets 3 of every 5 requests (uneven hardware)
upstream backend_servers {
    server api:3001  weight=3;
    server api2:3001 weight=1;
    server api3:3001 weight=1;
}

# Backup: api3 only receives traffic when the others are unavailable
upstream backend_servers {
    server api:3001;
    server api2:3001;
    server api3:3001 backup;
}
```

`ip_hash;` (mentioned in the Sarvin intro) pins each client IP to one backend, which helps when sessions are stored in server memory.

### Failover test

```bash
docker stop <project>-api2-1
docker ps                    # api2 gone
# refresh the page: responses now alternate api and api3
docker start <project>-api2-1
```

**Limitations.** All replicas on one VPS protect against a crashed container, not against the VPS failing. Round robin ignores request cost; `least_conn` handles uneven durations better. Nginx's default passive health checking takes a server out only after failed requests; active health checks are in commercial NGINX Plus.

**Cross-links.** Section 4 (headers forwarded to backends), section 8 (cache in front of the group).

## 8. Caching and compression

### HTTP caching (Sarvin)

**Define.** `proxy_cache` stores backend responses on disk and replays them (HTTP caching rules: RFC 9111).

**Problem.** Many users request the same page or API result; without a cache each request goes to the app and database again. The demo endpoint `/api/process-data` deliberately sleeps 7 seconds, so every click costs 7 seconds.

**Mechanism.** Declare a cache zone in the `http` context (a file in `conf.d` is already inside `http`), then enable it per location.

```nginx
proxy_cache_path /data/nginx/cache
                 keys_zone=api_cache:10m   # 10 MB of shared memory for keys
                 max_size=1g               # max disk size of the cache
                 inactive=30m              # drop items not used for 30 min
                 use_temp_path=off;        # write directly into the cache dir

server {
    # ...
    # cached endpoint (exact match)
    location = /api/process-data {
        proxy_pass http://backend_servers/process-data;
        proxy_cache api_cache;
        proxy_cache_valid 200 10m;         # cache 200 responses for 10 minutes
    }

    # everything else under /api is NOT cached
    location /api/ {
        proxy_pass http://backend_servers/;
    }
}
```

The container's cache directory must survive recreation, so mount a named volume:

```yaml
  nginx:
    volumes:
      - nginx_cache:/data/nginx/cache

volumes:
  nginx_cache:
```

Steps used: push, `git pull`, `docker compose down`, `docker compose up --build -d`, `docker exec <nginx> nginx -t`. First click takes about 7 seconds; the second is instant.

**The classic bug.** Caching all of `/api/` made a newly created user ("Sara") invisible, because `GET /api/users` kept returning the stale list. Fix: cache only endpoints whose response is identical for every user and may be stale for the TTL (the exact-match location above), and leave frequently changing or personalised ones such as `/api/users` uncached. A safe, idempotent method alone does not make a response cacheable (section 16).

Note on the exact-match location: with `=` you must give the real backend path in `proxy_pass` (here `/process-data`), because a URI in `proxy_pass` replaces the whole matched URI. Optional debugging addition, not in the sources: `add_header X-Cache-Status $upstream_cache_status;` shows HIT or MISS.

**Limitations.** Cached responses can leak per-user data when the shared cache ignores cookies or auth headers; bypass the cache and do not store such responses (section 16) rather than building per-user cache keys.

### Compression (gzip)

**Define.** The browser advertises supported encodings in `Accept-Encoding`; the server answers with `Content-Encoding: gzip` (RFC 9110; gzip is RFC 1952). Brotli (`br`, RFC 7932) is also advertised by browsers but needs an extra nginx module, which the sources only mention.

**Mechanism.** In the demo, a "big data" JSON response of about 9 to 10 MB shrank to roughly 1.6 MB with no application change.

```nginx
gzip on;
gzip_min_length 1000;                       # bytes; do not compress tiny responses
gzip_comp_level 5;                          # 1 (default, fast) to 9 (smallest, most CPU)
gzip_types text/css application/json application/javascript text/xml application/xml image/svg+xml;
```

`text/html` is always compressed once `gzip on` is set. Check it:

```bash
curl -s -o /dev/null -D - -H 'Accept-Encoding: gzip' https://example.com/api/big-data | grep -i content-encoding
```

In DevTools: request header `Accept-Encoding: gzip, deflate, br, zstd`, response header `Content-Encoding: gzip`.

**Limitations.** Level 9 shrinks the payload only slightly more than lower levels but costs noticeably more CPU and latency; the tutorial settles on level 5 as a balance. Do not compress already compressed formats (JPEG, PNG, MP4, ZIP).

**Cross-links.** Section 7 (the cache sits in front of the upstream group); RFC 9110 on `Vary: Accept-Encoding` for caches.

## 9. Security hardening

The proxy is a single entry point, so the whole attack surface is concentrated there; backends stay on a private network and are never published directly. These settings add layers; they do not replace a firewall or application-level security.

### Hide the version

```nginx
http {
    server_tokens off;     # Server: nginx (no version number)
}
```

On a host install this goes in `/etc/nginx/nginx.conf`, inside `http`; in the Docker setup the demo put it at the top of `default.conf`. Check with the response header `Server` in DevTools. Hiding the version is minor hardening, not a defence.

### Security headers

```nginx
add_header X-Content-Type-Options nosniff always;                       # no MIME sniffing
add_header X-Frame-Options SAMEORIGIN always;                           # or DENY: anti-clickjacking
add_header Referrer-Policy strict-origin-when-cross-origin always;      # limit referrer leakage
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;  # () = nobody may use it
```

| Header | Protects against | Note |
| --- | --- | --- |
| `X-Content-Type-Options: nosniff` | Browser executing a file as another type (a `.txt` holding script) | Trust server's `Content-Type` |
| `X-Frame-Options` | Clickjacking via `<iframe>` | `DENY`, or `SAMEORIGIN` if you embed your own pages |
| `Referrer-Policy` | Full previous URL sent to other sites |  |
| `Permissions-Policy` | Pages using location, mic, camera | Do not disable what your app needs |
| `Content-Security-Policy` | XSS and unwanted resource loading | See below |

Content-Security-Policy: the demo tried `add_header Content-Security-Policy "default-src 'self'";` and the page broke, because Bootstrap CSS came from a CDN. The tutorial removed it. A working policy must list every external origin you use (CDN, fonts, analytics, API, YouTube iframes). `X-Frame-Options` is largely superseded by CSP `frame-ancestors`.

Gotcha: `add_header` directives are inherited only if the current level defines none of its own. Declaring one `add_header` inside a `location` drops all those set at `server` level, so repeat them or use an `include`.

### Rate limiting

```nginx
http {
    # track by client IP, 10 MB of state, 5 requests per second
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

    server {
        location /api/ {
            limit_req zone=api_limit;          # excess requests are rejected (503 by default)
            proxy_pass http://backend_servers/;
        }
    }
}
```

Refreshing the page several times in a second made the API calls fail after the fifth. Optional additions not in the sources: `limit_req zone=api_limit burst=10 nodelay;` to tolerate short spikes and `limit_req_status 429;` for a conventional status. Rate limiting reduces abuse and simple HTTP floods; it does not stop distributed attacks, and clients behind one NAT share an IP.

### Block direct access by IP or unknown Host

```nginx
server {
    listen 80 default_server;      # catches any Host that matches no server_name
    server_name _;                  # placeholder name that matches no real domain
    return 444;                     # close connection without a response
}
```

HTTPS needs its own catch-all, otherwise `https://SERVER_IP` still reaches the app. A TLS server block normally requires a certificate; on nginx 1.19.4 or newer you can refuse the handshake instead (addition to the tutorial, which only said to add a port-443 block):

```nginx
server {
    listen 443 ssl default_server;
    server_name _;
    ssl_reject_handshake on;
}
```

The tutorial stresses that this is an nginx-level restriction. To truly hide the server, use a firewall too (for example `ufw allow 80,443/tcp`, `ufw allow OpenSSH`, `ufw enable`; the firewall commands are an addition, not from the sources).

**Cross-links.** Section 6 (HTTPS and redirects), section 4 (Mongo Express and other admin UIs should not be public: protect them with auth or an IP allowlist).

## 10. Architecture, request lifecycle, configuration internals

Sections 10 to 19 were added after the first pass to cover what the three tutorials skip: how nginx actually decides things, and what production needs. Examples follow the documented behaviour of nginx 1.24 to 1.28 and were not executed while writing; version requirements are stated where they exist. Always confirm with `nginx -V` and the docs of your version.

### Master and worker processes

**Define.** One *master* process reads and validates the configuration, binds the listening sockets and manages *workers*. Workers do all request handling.

**Problem.** Thread-per-connection servers spend memory and context switches on idle connections.

**Mechanism.** Each worker runs a non-blocking event loop (epoll on Linux, kqueue on BSD/macOS) and multiplexes thousands of connections, so one worker rarely blocks on I/O. On `nginx -s reload` (SIGHUP) the master parses the new config; if it is valid it starts new workers, then tells old workers to stop accepting and finish in-flight requests. If the config is invalid the old workers keep running.

```bash
ps -o pid,ppid,user,cmd -C nginx          # one master (root) + N workers (www-data/nginx)
sudo kill -HUP  $(cat /run/nginx.pid)     # same as nginx -s reload
sudo kill -QUIT $(cat /run/nginx.pid)     # graceful stop (finish requests)
sudo kill -TERM $(cat /run/nginx.pid)     # fast stop
sudo kill -USR1 $(cat /run/nginx.pid)     # reopen log files (used by logrotate)
sudo kill -USR2 $(cat /run/nginx.pid)     # start a new binary (zero-downtime upgrade)
```

**Limitations.** CPU-heavy work in a worker (large synchronous Lua/njs scripts, very large uncached gzip jobs) stalls every connection on that worker.

### Connection capacity (a correction to section 2)

The formula `worker_processes x worker_connections` gives the connection slots, not the number of clients you can serve. Each proxied request uses one slot for the client and one for the upstream connection, so a reverse proxy reaches roughly half of that figure. Every slot is also a file descriptor, so the OS limit matters:

```nginx
worker_processes auto;
worker_rlimit_nofile 65535;      # raises RLIMIT_NOFILE for workers; keep >= 2 x worker_connections

events {
    worker_connections 8192;
    multi_accept on;             # optional: accept all pending connections at once
}
```

```bash
# systemd services have their own limit; check and raise it
systemctl show nginx -p LimitNOFILE
sudo systemctl edit nginx       # add:  [Service]  LimitNOFILE=65535
cat /proc/$(pgrep -o -f 'nginx: master')/limits | grep -i 'open files'
```

Symptom of too few descriptors in `error.log`: `worker_connections are not enough` or `Too many open files`.

### Request lifecycle (the order nginx evaluates things)

1. **Accept** the TCP connection on a `listen` socket; for TLS, perform the handshake. The certificate is chosen by SNI (the server name sent in ClientHello).
2. **Select the `server` block** by listen address and port, then by the `Host` header against `server_name` (section 11).
3. **Server-level `rewrite`/`return`** directives run.
4. **Select the `location`** by the normalized URI (precedence rules in section 11).
5. **Location-level `rewrite`** runs; if a rewrite uses `last`, steps 4 and 5 repeat for the new URI (at most 10 times, then 500).
6. **Access limiting**: `limit_req`, `limit_conn` (pre-access phase), then `allow`/`deny`, `auth_basic`, `auth_request` (access phase, combined by `satisfy`).
7. **`try_files`** (pre-content phase).
8. **Content handler** produces the response: static file, `proxy_pass`, `fastcgi_pass`, `return`, ...
9. **Filters** transform the output (gzip, headers via `add_header`, chunking).
10. **Log** phase writes the access log.

This order explains common surprises: `limit_req` runs before authentication, so unauthenticated floods are still counted; `return` in a `server` block short-circuits any `location`; `add_header` is applied in the filter stage and therefore only on the responses nginx itself finishes.

### Configuration inheritance

Directives are inherited from outer to inner contexts (`http` > `server` > `location`), but the rule differs per directive:

| Kind | Behaviour | Examples |
| --- | --- | --- |
| Single-value | Inner value overrides outer value | `root`, `proxy_read_timeout`, `gzip_comp_level` |
| Array-type | Inherited only if the inner level defines none; one inner line discards all outer lines | `add_header`, `proxy_set_header`, `proxy_hide_header` |
| Not inherited | Must be set where used | `proxy_pass`, `fastcgi_pass`, `limit_req` (inherited by sub-locations, not across servers) |

```nginx
server {
    add_header X-Frame-Options SAMEORIGIN always;       # (A)
    location /api/ {
        add_header Cache-Control "no-store" always;     # defining any add_header here drops (A)
        proxy_pass http://backend_servers/;
    }
}
```

Fix: keep security headers in `snippets/security-headers.conf` and `include` it in each location that sets its own headers. The same applies to `proxy_set_header`: set `Host` and `X-Real-IP` again when a location adds its own header.

### Inspecting the effective configuration

```bash
sudo nginx -T                         # test + dump the full config with every include expanded
sudo nginx -T 2>/dev/null | grep -n 'server_name'   # where is a host defined?
sudo nginx -T 2>/dev/null | grep -nE 'listen\s+.*(80|443)'   # conflicting listeners
sudo nginx -t -c /tmp/candidate.conf  # test a different file without touching the live one
```

`nginx -T` is the first command to run when "my change has no effect": a stale symlink in `sites-enabled`, a `default` site still enabled, or a duplicate `server_name` are visible immediately. A warning `conflicting server name "example.com" on 0.0.0.0:80, ignored` means a second block was discarded.

### Modules and build options

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'module|with-'      # compiled-in modules and flags
ls /usr/lib/nginx/modules /etc/nginx/modules-enabled 2>/dev/null   # dynamic modules (Debian)
```

```nginx
# main context, before events/http: load a dynamic module
load_module modules/ngx_http_geoip2_module.so;
```

Static modules are built in (`--with-http_ssl_module`, `--with-http_v2_module`, `--with-stream`); dynamic ones are `.so` files loaded with `load_module`. A directive that is unknown at `nginx -t` ("unknown directive") usually means the module is not compiled in or not loaded. `ssl_preread`, `stream`, `http_v3_module`, `stub_status` and `gzip_static` each need their build flag; the official Docker images include most of them.

### Editions and versions

| Name | What it is | Notes |
| --- | --- | --- |
| nginx open source (stable / mainline) | The free server used throughout these notes | Stable branch gets fixes only; mainline gets features (such as `http2 on;` in 1.25.1) |
| NGINX Plus | Commercial F5 product | Active health checks, sticky cookies, `drain`, `slow_start`, cache purge, live API/dashboard |
| Third-party / community modules | Lua (OpenResty), ModSecurity connector, `ngx_cache_purge`, brotli | You compile or install them; check compatibility with your exact nginx version |
| Forks | Angie, freenginx | Same configuration language, different feature sets |

Each example in sections 10 to 19 states a version requirement when it has one.

## 11. Routing in depth

### Virtual hosting and server selection

One nginx can serve many sites. Selection is two-step: first the listen address and port, then the `Host` header (HTTP/1.1, RFC 9110) or `:authority` (HTTP/2) against `server_name`. For TLS, the certificate is chosen earlier, by SNI in the ClientHello (RFC 6066).

`server_name` match order: exact name, longest leading wildcard (`*.example.com`), longest trailing wildcard (`www.example.*`), first matching regex in file order, then the `default_server` of that listen socket (or the first server defined for it).

```nginx
# /etc/nginx/conf.d/sites.conf
server {
    listen 80 default_server;           # unknown Host or direct IP lands here
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

server {
    listen 80;
    listen [::]:80;                     # IPv6
    server_name example.com www.example.com;
    root /var/www/example;
}

server {
    listen 80;
    listen [::]:80;
    server_name blog.example.com;       # subdomain, separate site
    root /var/www/blog;
}

server {
    listen 80;
    server_name ~^(?<tenant>[a-z0-9-]+)\.app\.example\.com$;   # regex + named capture
    root /var/www/tenants/$tenant;
}
```

Test without touching DNS:

```bash
curl -i --resolve blog.example.com:80:127.0.0.1 http://blog.example.com/
curl -i -H 'Host: unknown.example.org' http://127.0.0.1/      # expect: connection closed (444)
```

### IPv6

Create an AAAA record next to every A record, add a `listen [::]:PORT;` line for each listener, and test both families. Without the AAAA/`[::]` pair, IPv6-only clients cannot reach you; with an AAAA record but no `[::]` listener they get timeouts.

```bash
curl -4 -I https://example.com/
curl -6 -I https://example.com/
dig +short AAAA example.com
ss -tlnp | grep -E ':(80|443)\b'      # look for both 0.0.0.0:443 and [::]:443
```

### Location matching precedence

**Define.** `location` picks the configuration that handles the request, from the normalized URI (percent-decoding, `//` merged, `.` and `..` resolved), without the query string.

**Mechanism.** The algorithm is fixed:

1. Check exact locations (`= /path`). If one matches, use it and stop.
2. Among prefix locations (no modifier, or `^~`), find the longest matching prefix.
3. If that longest prefix has `^~`, use it and stop (regexes are skipped).
4. Otherwise test regex locations (`~` case-sensitive, `~*` case-insensitive) in the order they appear in the file; the first match wins.
5. If no regex matches, use the longest prefix remembered in step 2.

```nginx
location = /                { return 200 "A exact root\n"; }
location /                  { return 200 "B generic prefix\n"; }
location /docs/             { return 200 "C prefix\n"; }
location ^~ /static/        { return 200 "D prefix, blocks regexes\n"; }
location ~* \.(png|jpg)$    { return 200 "E regex images\n"; }
location ~ ^/docs/.*\.pdf$  { return 200 "F regex pdf\n"; }
```

| Request | Winner | Why |
| --- | --- | --- |
| `/` | A | exact match |
| `/index.html` | B | only prefix `/` matches; no regex does |
| `/docs/guide.html` | C | longest prefix, no regex matches |
| `/docs/manual.pdf` | F | regex beats the plain prefix C; F is the first matching regex |
| `/static/logo.png` | D | `^~` stops the search before regexes (otherwise E would win) |
| `/img/cat.PNG` | E | `~*` is case-insensitive |

Nested locations are allowed for regexes and prefixes (`location /api/ { location ~ \.json$ { ... } }`) but each level repeats the algorithm; keep nesting shallow. Debug which block was used with `add_header X-Loc "api" always;` or a log field.

### Named locations and internal fallbacks

```nginx
location / {
    try_files $uri $uri/ @backend;     # file first, otherwise ask the app
}
location @backend {                    # named: unreachable directly from the client
    proxy_pass http://app:3000;
}

location /downloads/ {
    internal;                          # only reachable by internal redirects (X-Accel-Redirect, error_page)
    alias /srv/private/;
}
```

### URI vs file path variables

| Variable | Meaning | Example for `GET /a%20b/../c?x=1` |
| --- | --- | --- |
| `$request_uri` | original, unnormalized, with query string | `/a%20b/../c?x=1` |
| `$uri` | normalized, decoded, without query string; changes after rewrite | `/c` |
| `$args` / `$arg_x` | query string / one parameter | `x=1` / `1` |
| `$document_root$uri` | filesystem path built by `root` | `/var/www/c` |

Correction to section 3: `root` builds the file path as `root + full request URI`; `alias` replaces the matched location prefix with its value. With `location /fruits { root /srv/site; }` the file for `/fruits` is `/srv/site/fruits`; with `location /carbs { alias /srv/site/fruits; }` the file for `/carbs/x.html` is `/srv/site/fruits/x.html`. Always end `alias` with a slash when the location ends with one (`location /img/ { alias /data/img/; }`), or paths are mangled; this alias mistake ("off-by-slash") is a known directory-traversal source.

### Rewrite behaviour: flags, loops, query strings

```nginx
# flag last: stop this rewrite set, re-run location matching with the new URI
rewrite ^/old/(.*)$ /new/$1 last;

# flag break: stop rewriting, continue inside the CURRENT location (no new location search)
location /download/ {
    rewrite ^/download/(.*)$ /files/$1 break;
    root /srv;                      # serves /srv/files/<name>
}

# redirect (302) / permanent (301): send the client elsewhere
rewrite ^/blog/(\d+)$ /posts/$1 permanent;

# Query string: nginx appends the original args; end the replacement with ? to drop them
rewrite ^/search$ /find? permanent;       # /search?q=x -> /find
rewrite ^/search$ /find permanent;        # /search?q=x -> /find?q=x
```

Redirect loop symptoms: browser error `ERR_TOO_MANY_REDIRECTS`, or nginx logs `rewrite or internal redirection cycle` (10 internal redirects, answer 500). Typical cause: `return 301 https://$host$request_uri;` in a block that also receives HTTPS traffic from a TLS-terminating load balancer. Fix by redirecting only when `$scheme` or `X-Forwarded-Proto` says plain HTTP, or by splitting server blocks by port.

### Variables, `map`, and why `if` is dangerous

`map` (http context only) builds a variable from another, evaluated lazily and without the pitfalls of `if`:

```nginx
map $http_user_agent $is_bot {
    default 0;
    ~*(googlebot|bingbot|ahrefsbot) 1;
}

map $uri $new_uri {                   # large redirect table without 100 location blocks
    /old-page  /new-page;
    /about-us  /about;
    default    "";
}

server {
    if ($new_uri) { return 301 $new_uri; }      # safe use: if + return
    set $cache_ttl "1h";                        # set runs in the rewrite phase
}
```

Safe uses of `if` inside a `location` are only `return` and `rewrite ... last`. Mixing `if` with `proxy_pass`, `try_files`, `add_header` or other directives behaves unpredictably because `if` creates an implicit nested location ("If is evil" in the nginx wiki). Prefer `map`, `try_files`, and separate `location` blocks.

Useful built-in variables: `$host`, `$scheme`, `$remote_addr`, `$request_method`, `$request_id`, `$server_name`, `$status`, `$upstream_addr`, `$http_<header>`, `$cookie_<name>`, `$arg_<name>`.

### Single-page applications (React, Vue, Flutter Web)

Client-side routes such as `/dashboard` have no file on disk, so a reload gives 404 unless nginx falls back to `index.html`. Keep API errors separate, otherwise a mistyped `/api/xyz` returns HTML with status 200.

```nginx
server {
    listen 80;
    server_name app.example.com;
    root /var/www/app/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;       # file, directory, then SPA entry
    }

    location /api/ {                            # API never falls back to index.html
        proxy_pass http://backend_servers/;
    }

    location = /index.html {                    # shell must never be stale
        add_header Cache-Control "no-cache" always;
    }

    location /assets/ {                         # fingerprinted files (app.3f9a1c.js)
        expires 1y;
        add_header Cache-Control "public, immutable" always;
        try_files $uri =404;                    # missing asset is a real 404, not index.html
    }
}
```

### Custom error pages and maintenance mode

```nginx
server {
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html { root /var/www/errors; internal; }
    location = /50x.html { root /var/www/errors; internal; }

    location /api/ {
        proxy_pass http://backend_servers/;
        proxy_intercept_errors on;           # let nginx replace backend 4xx/5xx pages with error_page
    }
}

# maintenance switch: create /var/www/maintenance.flag to enable, remove to disable, then reload
map $maintenance $in_maintenance { default 0; "on" 1; }
server {
    set $maintenance "";
    if (-f /var/www/maintenance.flag) { set $maintenance "on"; }
    if ($in_maintenance) { return 503; }
    error_page 503 @maint;
    location @maint { root /var/www/errors; rewrite ^ /maintenance.html break; }
}
```

`error_page 404 =200 /index.html;` changes the status code; use it only deliberately (it hides real errors from monitoring).

### Filesystem permissions and 403 diagnosis

The worker user (`www-data` on Debian, `nginx` in the official image) needs execute (`x`) on every directory in the path and read (`r`) on the file. A 403 usually means one of these:

```bash
namei -l /var/www/my-first-site/index.html     # shows owner and mode of each path component
ps -o user= -C nginx | sort -u                  # which user runs workers
sudo chown -R www-data:www-data /var/www/site
sudo find /var/www/site -type d -exec chmod 755 {} \;
sudo find /var/www/site -type f -exec chmod 644 {} \;
sudo tail -n 20 /var/log/nginx/error.log        # "Permission denied" or "directory index ... is forbidden"
getenforce && sudo ausearch -m avc -ts recent   # SELinux: use chcon -t httpd_sys_content_t
```

| Error-log text | Cause |
| --- | --- |
| `directory index of "/var/www/x/" is forbidden` | no `index` file and `autoindex off` |
| `open() ... failed (13: Permission denied)` | mode/owner or SELinux/AppArmor |
| `open() ... failed (2: No such file or directory)` | wrong `root`/`alias`, usually missing or doubled path component |

Symlinks outside the root are followed by default; `disable_symlinks if_not_owner;` restricts that.

### Browser caching for static assets

Browser caching is separate from the proxy cache of section 8. The server tells the browser how long it may reuse a file (RFC 9111):

```nginx
location ~* \.(?:css|js|woff2|png|jpg|svg|webp)$ {
    expires 30d;                                    # sets Expires and Cache-Control: max-age
    add_header Cache-Control "public" always;
    etag on;                                        # default; enables 304 revalidation via If-None-Match
}
```

Best practice: fingerprint file names (`app.3f9a1c.js`) and use `Cache-Control: public, max-age=31536000, immutable`; keep `index.html` at `no-cache` so a new release is picked up. Verify:

```bash
curl -sI https://example.com/assets/app.3f9a1c.js | grep -iE 'cache-control|etag|expires|last-modified'
curl -sI -H 'If-None-Match: "<etag from above>"' https://example.com/assets/app.3f9a1c.js | head -1   # expect 304
```

**Limitations.** `expires` and `add_header Cache-Control` together can emit two `Cache-Control` headers; use one of them. **Cross-links.** Section 3 (basics), section 12 (proxying), RFC 9110 (Host, status codes), RFC 6066 (SNI).

## 12. Reverse proxy in depth

### Forwarded headers and trusting the proxy chain

**Define.** Behind nginx the backend sees nginx's IP and plain HTTP unless nginx passes the original values. The standard header is `Forwarded` (RFC 7239); in practice `X-Forwarded-For` (XFF) and `X-Forwarded-Proto` are used.

**Mechanism.** One reusable snippet keeps every location consistent:

```nginx
# /etc/nginx/snippets/proxy-headers.conf
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;   # appends $remote_addr to any incoming XFF
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Request-ID      $request_id;
```

```nginx
location /api/ {
    include snippets/proxy-headers.conf;     # repeat here: a proxy_set_header in the location would drop the server-level ones
    proxy_pass http://backend_servers/;
}
```

The application must be told to believe these headers, but only from nginx. Examples:

```javascript
// Express: trust exactly one proxy hop, so req.ip, req.protocol and secure cookies are correct
app.set('trust proxy', 1);
```

```python
# Django settings.py
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
USE_X_FORWARDED_HOST = True
```

**Limitations.** `$proxy_add_x_forwarded_for` keeps whatever the client sent, so XFF is spoofable at the first hop. If nginx is the edge, overwrite instead: `proxy_set_header X-Forwarded-For $remote_addr;`.

### Real client IP behind a CDN or load balancer

When nginx itself sits behind Cloudflare, an AWS ALB or another proxy, `$remote_addr` is that proxy. Restore the real client address only from sources you trust, with the `realip` module (`--with-http_realip_module`):

```nginx
# trusted proxies: your load balancer / CDN ranges (example private range + one Cloudflare range)
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 173.245.48.0/20;     # take the full, current list from the provider
real_ip_header   X-Forwarded-For;      # Cloudflare also offers CF-Connecting-IP
real_ip_recursive on;                  # skip trusted hops from the right to find the first untrusted address
```

Without `set_real_ip_from`, nginx ignores the header, which is the safe default. Any `set_real_ip_from 0.0.0.0/0` lets every client spoof its IP and defeats `limit_req`, `allow`/`deny` and your logs. Verify:

```bash
curl -s -H 'X-Forwarded-For: 1.2.3.4' https://example.com/whoami    # backend must NOT report 1.2.3.4 when sent from an untrusted source
```

If the front proxy speaks PROXY protocol instead, see section 19.

### Timeouts

| Directive | Default | Meaning | Typical tuning |
| --- | --- | --- | --- |
| `proxy_connect_timeout` | 60s | time to establish the TCP connection to the backend | `5s` (fail fast to the next upstream) |
| `proxy_send_timeout` | 60s | max gap between two writes of the request to the backend | `60s`; raise for slow uploads |
| `proxy_read_timeout` | 60s | max gap between two reads of the response; not the total time | `60s`; raise for long reports, `1h` for SSE |
| `send_timeout` | 60s | gap between two writes to the client | `30s` |
| `client_header_timeout` / `client_body_timeout` | 60s | time to receive request header / body gaps from the client | `10s` against slow clients |
| `keepalive_timeout` | 75s | idle client connection kept open | `30s` to `65s` |

```nginx
location /api/ {
    proxy_connect_timeout 5s;
    proxy_send_timeout    30s;
    proxy_read_timeout    30s;      # backend slower than 30s between bytes -> 504 Gateway Timeout
    proxy_pass http://backend_servers/;
}
```

Rule: nginx timeouts must be longer than the backend's own request timeout (otherwise the client gets a 504 while the app still works), and shorter than any load balancer or CDN in front of nginx (otherwise that layer answers first).

### Request and response buffering

By default nginx reads the whole backend response into buffers (then temp files) and sends it at the client's speed, so slow clients do not hold backend workers. It also buffers the client's request body before contacting the backend.

```nginx
location /api/ {
    proxy_buffering on;                 # default
    proxy_buffer_size 8k;               # first part: response headers
    proxy_buffers 16 16k;               # per connection
    proxy_busy_buffers_size 32k;
    proxy_max_temp_file_size 256m;      # 0 = never spill to disk
    proxy_request_buffering on;         # default; off streams uploads straight to the backend
    proxy_pass http://backend_servers/;
}
```

Turn `proxy_buffering off` only for streaming responses (section 13). Turn `proxy_request_buffering off` for very large uploads or when the backend wants to start processing at once; the cost is that a slow client now occupies a backend connection, and nginx can no longer retry the request on another upstream.

### Uploads and large requests

```nginx
http {
    client_max_body_size 1m;                 # default; larger body -> 413 Request Entity Too Large
}
server {
    location /upload/ {
        client_max_body_size 200m;            # raise only where needed
        client_body_timeout 120s;             # time allowed between two reads of the body
        client_body_buffer_size 256k;         # in-memory; larger bodies go to client_body_temp_path
        client_body_temp_path /var/cache/nginx/client_temp;
        proxy_request_buffering off;          # optional: stream
        proxy_pass http://backend_servers/upload/;
    }
}
```

Check free space of the temp path (`df -h /var/cache/nginx`): buffered uploads of 200 MB times many concurrent users fill a small disk. The backend may have its own limit (Express `bodyParser` limit, PHP `upload_max_filesize` and `post_max_size`); the smallest limit wins and the error text tells you which layer rejected the request (HTML from nginx, JSON from the app).

```bash
dd if=/dev/zero of=/tmp/50m.bin bs=1M count=50
curl -sS -o /dev/null -w '%{http_code}\n' -F file=@/tmp/50m.bin https://example.com/upload/   # 413 before the fix, 200 after
```

### Upstream connection reuse (keepalive)

Two different keepalives exist. Client-side keepalive (`keepalive_timeout`) is between browser and nginx. Upstream keepalive is between nginx and the backend; by default nginx opens a new TCP connection per proxied request and closes it.

```nginx
upstream backend_servers {
    server api:3001;
    server api2:3001;
    keepalive 32;                  # idle connections cached PER WORKER
    keepalive_requests 1000;       # requests per upstream connection
    keepalive_timeout 60s;         # (nginx >= 1.15.3) idle time of an upstream connection
}

location /api/ {
    proxy_http_version 1.1;               # HTTP/1.0 default cannot keep connections
    proxy_set_header Connection "";       # remove "Connection: close"
    proxy_pass http://backend_servers/;
}
```

The backend's keepalive timeout must be longer than nginx's idle time, or the backend closes a connection nginx is about to reuse, giving sporadic 502 with `upstream prematurely closed connection`. Node's default `server.keepAliveTimeout` is 5 s, so set it higher (for example 65 s) when you enable upstream keepalive.

### Backend redirects and cookies

A backend that issues `Location: http://api:3001/login` leaks the internal name. `proxy_redirect` rewrites `Location` and `Refresh` headers; the cookie directives rewrite `Set-Cookie`:

```nginx
location /app/ {
    proxy_pass http://app:3000/;
    proxy_redirect http://app:3000/ /app/;            # Location: http://app:3000/x -> /app/x
    proxy_cookie_path / /app/;                        # Set-Cookie ... Path=/ -> Path=/app/
    proxy_cookie_domain internal.local example.com;   # Domain=internal.local -> example.com
}
```

If the app generates absolute URLs from the scheme it sees, pass `X-Forwarded-Proto` and make the app honour it (above), otherwise users are redirected from HTTPS to `http://`, creating mixed-content errors or loops.

### CORS

CORS (Fetch standard; preflight is an `OPTIONS` request) is decided by the response headers. Choose one owner for it: the application or nginx, never both, or the browser rejects duplicated `Access-Control-Allow-Origin` values.

```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com"     $http_origin;
    "https://staging.example.com" $http_origin;
}

location /api/ {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin  $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
        add_header Access-Control-Max-Age 86400 always;
        add_header Vary Origin always;
        return 204;
    }
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Access-Control-Allow-Credentials true always;   # only valid with an explicit origin, never "*"
    add_header Vary Origin always;
    proxy_pass http://backend_servers/;
}
```

Check the preflight:

```bash
curl -i -X OPTIONS https://example.com/api/users \
  -H 'Origin: https://app.example.com' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
# expect 204 + exactly one Access-Control-Allow-Origin: https://app.example.com
```

The `if ... return` form is one of the safe uses of `if` (section 11). When the backend already sets CORS headers, hide theirs with `proxy_hide_header Access-Control-Allow-Origin;` before adding your own.

**Cross-links.** Section 13 for long-lived connections, section 14 for TLS and access control, section 15 for retries, RFC 7239.

## 13. Long-lived connections and other backend types

### WebSockets

**Define.** A WebSocket (RFC 6455) starts as an HTTP/1.1 request with `Upgrade: websocket` and `Connection: Upgrade`, then the same TCP connection carries framed messages in both directions.

**Problem.** `Upgrade` and `Connection` are hop-by-hop headers: nginx does not forward them, and it speaks HTTP/1.0 to backends by default, so the handshake fails (browser: `WebSocket connection failed`, nginx log: 400 or 200 without upgrade).

**Mechanism.** Use HTTP/1.1 to the backend and pass the upgrade headers. The `map` sets `Connection: close` for ordinary requests and `upgrade` for WebSocket ones, so one location serves both:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream ws_backend {
    server app:3000;
}

server {
    listen 443 ssl;
    server_name example.com;

    location /socket.io/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host       $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout  3600s;     # idle time with no frames before nginx closes (default 60s)
        proxy_send_timeout  3600s;
    }
}
```

Test:

```bash
# handshake only: expect "HTTP/1.1 101 Switching Protocols"
curl -i -N https://example.com/socket.io/?EIO=4\&transport=websocket \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ=='

# interactive client
npx wscat -c wss://example.com/socket.io/?EIO=4\&transport=websocket
```

**Limitations.** Idle WebSockets are closed after `proxy_read_timeout`; send application-level ping frames more often than that (every 25 to 30 s is common), and make clients reconnect with back-off. To test reconnects: `sudo nginx -s reload` (old workers keep existing sockets until `worker_shutdown_timeout`; set it, for example `worker_shutdown_timeout 60s;`, so deployments do not hang for hours). With several backends, Socket.IO long-polling needs affinity (`ip_hash;` or `hash $cookie_io consistent;`); pure WebSocket does not.

### Server-Sent Events (SSE) and streaming

SSE is a long-lived HTTP response with `Content-Type: text/event-stream`. Response buffering would hold the events back until a buffer fills, so disable it for the streaming location.

```nginx
location /events {
    proxy_pass http://app:3000;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;             # send each chunk immediately
    proxy_cache off;
    gzip off;                        # gzip buffers output; keep streams uncompressed
    proxy_read_timeout 1h;
    chunked_transfer_encoding on;    # default for HTTP/1.1 responses
}
```

The backend can also opt out per response with the header `X-Accel-Buffering: no`. Send a comment line (`: ping\n\n`) every 15 to 30 s as a heartbeat so that proxies and load balancers in the path do not drop the idle connection. Test:

```bash
curl -N -H 'Accept: text/event-stream' https://example.com/events     # -N: no client-side buffering
```

If events arrive in bursts instead of one by one, buffering is still on somewhere (nginx, a CDN, or the app framework's gzip middleware).

### gRPC

gRPC runs over HTTP/2 (RFC 9113), so the listener must speak HTTP/2 and the backend is reached with `grpc_pass`.

```nginx
server {
    listen 443 ssl;
    http2 on;                                  # nginx >= 1.25.1; older: listen 443 ssl http2;
    server_name grpc.example.com;
    ssl_certificate     /etc/ssl/grpc.crt;
    ssl_certificate_key /etc/ssl/grpc.key;

    location /helloworld.Greeter/ {             # /<package>.<Service>/ routes by service
        grpc_pass grpc://greeter:50051;         # plaintext HTTP/2 to the backend; grpcs:// for TLS
        grpc_read_timeout 300s;
        error_page 502 = /error502grpc;
    }
    location = /error502grpc {
        internal;
        default_type application/grpc;
        add_header grpc-status 14;              # UNAVAILABLE
        add_header content-length 0;
        return 204;
    }
}
```

```bash
grpcurl -insecure grpc.example.com:443 list     # needs server reflection, or pass -proto file
```

Common failures: listener without HTTP/2 (`connection closed before server preface`), `proxy_pass` used instead of `grpc_pass`, and a missing TLS layer on a plaintext listener (gRPC clients refuse it unless configured).

### PHP-FPM (FastCGI)

```nginx
server {
    listen 80;
    server_name php.example.com;
    root /var/www/app/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;      # front-controller frameworks (Laravel, Symfony)
    }

    location ~ \.php$ {
        try_files $uri =404;                                # never pass a non-existent script to PHP
        include fastcgi_params;                             # standard CGI variables
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;         # or 127.0.0.1:9000 or php-fpm:9000 in Docker
        fastcgi_read_timeout 60s;
    }

    location ~ /\.(?!well-known/) { deny all; }              # .git, .env, .htaccess
}
```

Safety notes: the `try_files $uri =404;` line prevents the classic `/upload/evil.jpg/x.php` path-info execution trick; also keep `cgi.fix_pathinfo=0` in `php.ini`, and do not let users upload files into a directory that PHP can execute. On Debian, `include snippets/fastcgi-php.conf;` bundles these steps. In Docker, nginx and PHP-FPM need the same code at the same path (mount one volume into both), because `SCRIPT_FILENAME` is a path inside the PHP container.

`uwsgi_pass` (`include uwsgi_params;`) and `scgi_pass` follow the same pattern for Python/uWSGI and SCGI applications.

### Unix-domain sockets

A backend on the same host can listen on a socket file instead of a TCP port: no port to expose, file permissions as access control, slightly lower overhead.

```nginx
upstream app {
    server unix:/run/app/app.sock;
    keepalive 16;
}
location / {
    proxy_pass http://app;
}
# or directly, note the colon after the socket path:
location /direct/ { proxy_pass http://unix:/run/app/app.sock:/; }
```

```bash
ls -l /run/app/app.sock                       # nginx worker user needs rw on the socket and x on /run/app
sudo -u www-data curl --unix-socket /run/app/app.sock http://localhost/healthz
```

Usual failure: `connect() to unix:/run/app/app.sock failed (13: Permission denied)`; add the nginx user to the app's group or set `SocketMode`/`umask` in the app.

### Protected downloads with X-Accel-Redirect

The application authorizes the user; nginx streams the file efficiently (sendfile, range requests, no app worker tied up).

```nginx
location /protected/ {
    internal;                          # clients cannot request /protected/... directly
    alias /srv/private-files/;         # trailing slash on both
}
location /download/ {
    proxy_pass http://app:3000;        # app checks the session and decides
}
```

```javascript
// Express handler for GET /download/:id after the permission check
app.get('/download/:id', requireLogin, (req, res) => {
  res.set('X-Accel-Redirect', `/protected/${req.params.id}.pdf`);   // internal URI
  res.set('Content-Type', 'application/pdf');
  res.set('Content-Disposition', `attachment; filename="report-${req.params.id}.pdf"`);
  res.end();
});
```

Validate `req.params.id` strictly (digits only); a path containing `../` would be normalized by nginx, but the application must never construct internal URIs from raw user input.

**Cross-links.** Section 12 (timeouts, buffering, keepalive), section 15 (deployments and `worker_shutdown_timeout`), RFC 6455 (WebSocket), RFC 9113 (HTTP/2).

## 14. TLS, certificate renewal, authentication and access control

### Certificate fundamentals

- A certificate binds a public key to names. Browsers check the **Subject Alternative Name (SAN)** list for the host; the old Common Name is ignored. `example.com` and `www.example.com` are two names and both must be listed.
- The server must send the **chain** (leaf plus intermediate CA certificates). Let's Encrypt gives `fullchain.pem` (use this for `ssl_certificate`) and `cert.pem` (leaf only; causes "unable to verify the first certificate" on some clients). Clients carry the root CAs in their trust store.
- The **private key** (`privkey.pem`) must be readable only by root (`chmod 600`); nginx's master process reads it as root before workers drop privileges. In Docker mount it read-only.

```bash
# what is actually deployed?
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# does the key match the certificate? both hashes must be identical
openssl x509 -in fullchain.pem -noout -pubkey | openssl sha256
openssl pkey -in privkey.pem -pubout | openssl sha256

# full chain as the server sends it (look for 'Verification: OK' / 'Verify return code: 0')
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null | grep -E 's:|i:|Verify return'
```

### ACME challenge methods

ACME (RFC 8555) proves control of a name before a CA issues a certificate.

| Method | Needs | Wildcard | Good for |
| --- | --- | --- | --- |
| HTTP-01 | Port 80 reachable from the internet; a file under `/.well-known/acme-challenge/` | No | Public web servers |
| DNS-01 | A TXT record `_acme-challenge.<name>`; DNS API credentials | Yes | Wildcards (`*.example.com`), servers behind firewalls, internal names |
| TLS-ALPN-01 | Port 443 | No | When port 80 is blocked (certbot does not support it natively) |

```bash
# DNS-01 with Cloudflare (wildcard); the token needs Zone:DNS:Edit only
echo 'dns_cloudflare_api_token = <TOKEN>' > cloudflare.ini && chmod 600 cloudflare.ini
docker run --rm -v /etc/letsencrypt:/etc/letsencrypt -v $PWD/cloudflare.ini:/cloudflare.ini:ro \
  certbot/dns-cloudflare certonly --dns-cloudflare --dns-cloudflare-credentials /cloudflare.ini \
  -d example.com -d '*.example.com' --email you@example.com --agree-tos --no-eff-email
```

### Automatic renewal when nginx runs in Docker (completes section 6)

**Problem.** Certificates issued by `certbot --nginx` on the host record `authenticator = nginx` in `/etc/letsencrypt/renewal/<name>.conf`. After nginx moves into a container that plugin cannot reach it, and a `--standalone` renewal fails because the container owns port 80. Also, a first start fails when `ssl_certificate` points at files that do not exist yet.

**Mechanism.** Use the **webroot** method: nginx serves the challenge file from a shared directory, a certbot container renews in a loop, and nginx reloads periodically so it picks up new certificates. Both containers share the same `/etc/letsencrypt` bind mount (certificates live on the host) and one webroot volume.

`nginx/default.conf` (HTTP block):

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;            # certbot writes the token file here
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
```

`docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:1.28-alpine            # pin a version; check the current stable tag
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - certbot_webroot:/var/www/certbot:ro
    # reload every 6 h so renewed certificates are loaded, then run nginx in the foreground
    command: >
      /bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & exec nginx -g "daemon off;"'
    restart: unless-stopped

  certbot:
    image: certbot/certbot
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
      - certbot_webroot:/var/www/certbot
    # try renewal twice a day; certbot only renews certificates within 30 days of expiry
    entrypoint: >
      /bin/sh -c 'trap exit TERM; while :; do certbot renew --webroot -w /var/www/certbot --quiet; sleep 12h & wait $${!}; done'
    restart: unless-stopped

volumes:
  certbot_webroot:
```

`$${!}` is Compose's escape for the shell variable `$!`. Certificates live on the host, so each container mounts `/etc/letsencrypt` once; nginx mounts it read-only and certbot read-write.

**First issuance (bootstrap).** Start nginx with only the HTTP server block, then request the certificate with webroot, then add the 443 block:

```bash
docker compose up -d nginx                                   # HTTP-only default.conf at this point
docker compose run --rm --entrypoint certbot certbot certonly --webroot -w /var/www/certbot \
  -d example.com -d www.example.com --email you@example.com --agree-tos --no-eff-email
# now add the listen 443 ssl server block, then:
docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload
```

**Migrating certificates issued with `certbot --nginx` on the host.** Re-issue once with webroot so the renewal file stops using the nginx authenticator:

```bash
docker compose run --rm --entrypoint certbot certbot certonly --webroot -w /var/www/certbot \
  --cert-name example.com -d example.com -d www.example.com --force-renewal
grep -E 'authenticator|webroot' /etc/letsencrypt/renewal/example.com.conf   # expect authenticator = webroot
```

**Test the whole path before you need it.**

```bash
docker compose run --rm --entrypoint certbot certbot renew --dry-run     # uses the staging CA; must end with "all simulated renewals succeeded"
curl -s http://example.com/.well-known/acme-challenge/test -o /dev/null -w '%{http_code}\n'   # 404 from nginx = location reachable
docker compose exec nginx nginx -T | grep -A2 acme-challenge
```

Host-install alternative (no Docker): `sudo systemctl list-timers | grep certbot` shows the packaged renewal timer; add a deploy hook so nginx reloads after each renewal: `echo 'systemctl reload nginx' | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh && sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh`.

**Alerting.** Do not rely on CA reminder e-mails (Let's Encrypt announced the end of expiry-notification e-mails in 2025; verify the current policy). Check expiry yourself:

```bash
#!/bin/sh
# cert-check.sh: exit 1 if the certificate expires within 14 days; run from cron or your monitor
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -checkend $((14*24*3600)) || { echo "cert expires soon"; exit 1; }
```

**Limitations.** Webroot needs port 80 open from the internet; use DNS-01 otherwise. Two instances behind one load balancer each need the challenge reachable, or use DNS-01. Certificate lifetimes are shrinking across the industry, so the renewal path matters more every year.

### TLS protocol settings

```nginx
ssl_protocols TLSv1.2 TLSv1.3;                 # RFC 8446 (1.3), RFC 5246 (1.2); disable SSLv3, TLS 1.0, 1.1
ssl_prefer_server_ciphers off;                 # modern clients choose well; TLS 1.3 ignores this
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_session_cache   shared:SSL:10m;            # about 40,000 sessions; avoids full handshakes
ssl_session_timeout 1d;
ssl_session_tickets off;                       # tickets reuse long-lived keys; off keeps forward secrecy simple
```

This follows the Mozilla "intermediate" profile; regenerate it from Mozilla's SSL Configuration Generator when you upgrade. Verify what is actually negotiated:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3 -brief </dev/null    # expect TLSv1.3
openssl s_client -connect example.com:443 -servername example.com -tls1_1 </dev/null 2>&1 | grep -i 'alert\|handshake failure'   # must fail
nmap --script ssl-enum-ciphers -p 443 example.com
docker run --rm drwetter/testssl.sh https://example.com                  # full report
```

### HSTS (HTTP Strict Transport Security, RFC 6797)

HSTS tells browsers to use only HTTPS for the host for `max-age` seconds, removing the unprotected first redirect on later visits. It is hard to undo, so roll it out gradually:

| Step | Header | Wait |
| --- | --- | --- |
| 1 | `max-age=300` | a day |
| 2 | `max-age=604800` (1 week) | a week |
| 3 | `max-age=15552000; includeSubDomains` (6 months) | after checking every subdomain serves valid HTTPS |
| 4 | `max-age=63072000; includeSubDomains; preload` (2 years), then submit to hstspreload.org | only if you accept that removal from browser preload lists takes months |

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;   # inside the 443 server block only
```

If HTTPS later breaks (expired certificate), HSTS-cached browsers show a hard error without a click-through option: another reason to test renewal first. Never send the header over plain HTTP; browsers ignore it there.

### HTTPS to the backends (upstream TLS)

When traffic between nginx and a backend crosses a network you do not trust, encrypt it and verify the backend's certificate. By default nginx does **not** verify upstream certificates, so an encrypted but unverified hop still allows impersonation.

```nginx
location /secure-api/ {
    proxy_pass https://api.internal.example.com;
    proxy_ssl_server_name on;                        # send SNI to the backend
    proxy_ssl_name api.internal.example.com;         # name used for SNI and verification
    proxy_ssl_verify on;
    proxy_ssl_verify_depth 2;
    proxy_ssl_trusted_certificate /etc/nginx/certs/internal-ca.pem;   # CA that signed the backend certificate
    proxy_ssl_protocols TLSv1.2 TLSv1.3;
    # optional: nginx authenticates itself to the backend (mutual TLS)
    proxy_ssl_certificate     /etc/nginx/certs/nginx-client.crt;
    proxy_ssl_certificate_key /etc/nginx/certs/nginx-client.key;
}
```

Diagnose with `openssl s_client -connect api.internal.example.com:443 -servername api.internal.example.com -CAfile internal-ca.pem`. Typical log line when verification fails: `upstream SSL certificate verify error: (20:unable to get local issuer certificate)`.

### Mutual TLS (client certificates)

With mTLS the client also presents a certificate signed by a CA you trust: strong access control for admin tools and service-to-service calls.

```bash
# 1. private CA
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 -keyout ca.key -out ca.crt -subj "/CN=Internal CA"
# 2. client key + signing request + certificate
openssl req -newkey rsa:2048 -nodes -keyout alice.key -out alice.csr -subj "/CN=alice"
openssl x509 -req -in alice.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -out alice.crt
```

```nginx
server {
    listen 443 ssl;
    server_name admin.example.com;
    ssl_certificate     /etc/letsencrypt/live/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;

    ssl_client_certificate /etc/nginx/certs/ca.crt;    # CA(s) allowed to sign client certificates
    ssl_verify_client on;                              # reject clients without a valid certificate (400)
    ssl_verify_depth 2;

    location / {
        proxy_set_header X-Client-CN $ssl_client_s_dn;           # identity for the backend
        proxy_set_header X-Client-Verify $ssl_client_verify;     # SUCCESS when valid
        proxy_pass http://admin-ui:8080;
    }
}
```

```bash
curl --cert alice.crt --key alice.key https://admin.example.com/       # 200
curl https://admin.example.com/                                        # 400 No required SSL certificate was sent
```

Use `ssl_verify_client optional;` plus a check on `$ssl_client_verify` when only some paths need a certificate. Revoke with `ssl_crl /etc/nginx/certs/ca.crl;`.

### HTTP Basic authentication

Basic authentication sends `user:password` base64-encoded (not encrypted), so use it only over HTTPS. It is the quickest gate for admin interfaces such as the Mongo Express page in the demo.

```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin            # -c creates the file; omit it for further users
# without apache2-utils:
printf 'admin:%s\n' "$(openssl passwd -apr1)" | sudo tee -a /etc/nginx/.htpasswd
sudo chown root:www-data /etc/nginx/.htpasswd && sudo chmod 640 /etc/nginx/.htpasswd
```

```nginx
location /mongo/ {
    auth_basic           "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;     # in Docker: mount it read-only
    proxy_set_header Authorization "";             # do not leak the nginx credentials; avoids a clash if the app has its own basic auth
    proxy_pass http://express:8081;
}
```

```yaml
  nginx:
    volumes:
      - ./nginx/.htpasswd:/etc/nginx/.htpasswd:ro     # keep this file out of Git or encrypt it (SOPS, Docker secrets)
```

```bash
curl -i https://example.com/mongo/                       # 401 + WWW-Authenticate: Basic
curl -i -u admin:'S3cret!' https://example.com/mongo/    # 200
```

Mongo Express enables its own basic auth by default and reads the same `Authorization` header; that is why the line clearing it above matters (alternatively disable one of the two layers).

### IP access control and combining it with authentication

```nginx
location /admin/ {
    allow 203.0.113.10;            # office
    allow 10.0.0.0/8;              # VPN
    deny  all;                     # rules are checked top to bottom, first match wins
    proxy_pass http://admin-ui:8080;
}

location /internal/ {
    satisfy any;                   # allow EITHER a trusted IP OR valid credentials (satisfy all = both)
    allow 10.0.0.0/8;
    deny  all;
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://app:3000;
}
```

`allow`/`deny` evaluate `$remote_addr`; behind a proxy they are only meaningful after the `realip` setup of section 12, otherwise every client appears as the proxy.

### External authentication with auth\_request

nginx can ask another service whether a request may proceed (module `ngx_http_auth_request_module`, check `nginx -V` for `--with-http_auth_request_module`). A 2xx answer allows, 401 or 403 denies.

```nginx
location /api/ {
    auth_request /_auth;                                  # subrequest before proxying
    auth_request_set $auth_user $upstream_http_x_user;    # capture a header from the auth reply
    proxy_set_header X-User $auth_user;
    error_page 401 = @login_redirect;
    proxy_pass http://backend_servers/;
}

location = /_auth {
    internal;
    proxy_pass http://auth-service:9000/verify;
    proxy_pass_request_body off;                          # the auth service only needs the headers
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
    proxy_set_header X-Original-Method $request_method;
}

location @login_redirect { return 302 /login?next=$request_uri; }
```

```javascript
// Express auth service: 204 = allow, 401 = deny
app.get('/verify', (req, res) => {
  const token = (req.get('Authorization') || '').replace(/^Bearer /, '');
  const user = verifyJwt(token);                       // your verification, e.g. jsonwebtoken
  if (!user) return res.sendStatus(401);
  res.set('X-User', user.sub).sendStatus(204);
});
```

Every protected request now costs one extra subrequest; keep the auth service fast and local, and consider `proxy_cache` for its answers keyed on the token for a few seconds.

### Protecting sensitive files

```nginx
autoindex off;                                              # default; never list directories on public sites

location ~ /\.(?!well-known/) { deny all; }                 # .git, .env, .htpasswd, .svn (keep ACME paths)
location ~* \.(?:env|ini|log|bak|old|orig|sql|swp|conf|yml|yaml)$ { deny all; }
location ~* ^/(?:composer\.(?:json|lock)|package(?:-lock)?\.json|Dockerfile|docker-compose\.ya?ml)$ { deny all; }
```

Test: `curl -I https://example.com/.git/config` and `curl -I https://example.com/.env` must return 403 or 404, never 200. Best practice: keep these files out of the web root entirely; the rules are a second line of defence. Use `return 404;` instead of `deny all;` if you do not want to confirm that the file exists.

### Content-Security-Policy rollout

The demo's `default-src 'self'` broke the page because it used a CDN stylesheet. A policy is built from what the page really loads. Start in report-only mode so nothing breaks while you collect violations:

```nginx
add_header Content-Security-Policy-Report-Only "default-src 'self'; style-src 'self' https://cdn.jsdelivr.net; script-src 'self' https://cdn.jsdelivr.net; img-src 'self' data:; font-src 'self' https://cdn.jsdelivr.net; connect-src 'self'; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; report-uri /csp-report" always;

location = /csp-report {
    access_log /var/log/nginx/csp.log;
    proxy_pass http://app:3000/csp-report;     # or: return 204;
}
```

Watch the browser console and `csp.log` for a week, add the missing sources, then rename the header to `Content-Security-Policy`. Notes: inline scripts and styles need a per-response nonce or a hash (`'sha256-...'`), not `'unsafe-inline'`; generating a nonce is the application's job (nginx can only substitute `$request_id` through `sub_filter`, which is fragile). `frame-ancestors` supersedes `X-Frame-Options` in modern browsers. A CSP is defence in depth against XSS, not a replacement for output encoding.

### Abuse controls beyond rate limiting

```nginx
http {
    # concurrent connections per IP
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    limit_req_zone  $binary_remote_addr zone=api_limit:10m rate=5r/s;

    # slow-client defences (Slowloris style)
    client_header_timeout 10s;
    client_body_timeout   10s;
    send_timeout          30s;
    reset_timedout_connection on;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;          # larger header -> 400/414
    client_max_body_size 1m;

    server {
        location /api/ {
            limit_conn conn_limit 20;                      # max 20 open connections per IP
            limit_req  zone=api_limit burst=10 nodelay;    # tolerate short spikes of 10
            limit_req_status  429;                         # default is 503
            limit_conn_status 429;
            limit_req_log_level warn;
            proxy_pass http://backend_servers/;
        }

        location /login {
            limit_req zone=api_limit burst=3;              # no nodelay: excess is queued and spaced out
            proxy_pass http://app:3000;
        }
    }
}
```

Roll out safely with `limit_req_dry_run on;` (nginx >= 1.17.1): it logs what would be rejected without rejecting. Exempt trusted IPs (monitoring, office) by keying on a variable that is empty for them, because an empty key is not counted:

```nginx
geo $limited { default 1; 10.0.0.0/8 0; 203.0.113.10 0; }
map $limited $limit_key { 0 ""; 1 $binary_remote_addr; }
limit_req_zone $limit_key zone=api_limit:10m rate=5r/s;
```

Test: `for i in $(seq 1 30); do curl -s -o /dev/null -w '%{http_code} ' https://example.com/api/users; done` shows `200` and then `429`. Rate limiting does not stop distributed attacks and clients behind one NAT share an IP.

### WAF and what nginx alone cannot do

nginx does not inspect request bodies for attack patterns (SQL injection, XSS payloads, known CVE exploits). Options are an additional engine (ModSecurity v3 with the OWASP Core Rule Set through the nginx connector; F5 NGINX App Protect with NGINX Plus) or a managed WAF in front (Cloudflare, AWS WAF). Start any WAF in detection-only mode, review false positives, then enforce:

```nginx
load_module modules/ngx_http_modsecurity_module.so;     # main context; module must match your exact nginx version
http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;   # in the rules file: SecRuleEngine DetectionOnly, then On
}
```

Check the maintenance state and nginx-version support of the connector you choose; third-party modules often lag behind nginx releases.

### Logging privacy

Logs are personal data under GDPR (IP addresses count) and often contain secrets:

- Never add `$http_authorization`, `$http_cookie` or `$http_x_api_key` to a `log_format`.
- Query strings may carry tokens (`?token=...`, password reset links). Log the path only.
- Anonymize or truncate IPs when you do not need them, and set a retention period (section 17).

```nginx
map $request_uri $loggable_uri {
    ~^(?<path>[^?]*)  $path;       # strip the query string
    default           $request_uri;
}
map $remote_addr $remote_addr_anon {
    ~(?P<ip>\d+\.\d+\.\d+)\.\d+  $ip.0;          # 203.0.113.77 -> 203.0.113.0
    ~(?P<ip>[^:]+:[^:]+):         $ip::;         # IPv6: keep the first 2 groups
    default                       0.0.0.0;
}
log_format privacy '$remote_addr_anon - [$time_local] "$request_method $loggable_uri" $status $body_bytes_sent "$http_user_agent"';
access_log /var/log/nginx/access.log privacy;
```

**Cross-links.** Section 6 (basic HTTPS), section 9 (headers, rate limit), section 12 (real IP), RFC 8555 (ACME), RFC 6797 (HSTS), RFC 8446 (TLS 1.3), RFC 7235 (HTTP authentication, superseded by RFC 9110).

## 15. Availability, upstream behaviour and deployments

### Passive health checks (open-source nginx)

**Define.** Nginx marks a backend as unavailable based on the outcome of real client requests; it does not probe backends by itself.

**Mechanism.** Two parameters on each `server` line of the `upstream` block:

- `max_fails=N` (default 1): number of failed attempts within `fail_timeout` that mark the server unavailable. `max_fails=0` turns checking off.
- `fail_timeout=T` (default 10s): both the window in which failures are counted and the time the server is then skipped. After that nginx tries it again with one request.

```nginx
upstream backend_servers {
    zone backend_servers 64k;                      # shared memory: all workers see the same failure counters
    server api:3001  max_fails=3 fail_timeout=30s;
    server api2:3001 max_fails=3 fail_timeout=30s;
    server api3:3001 backup;
    # server api4:3001 down;                      # administratively removed
}
```

What counts as a failure is decided by `proxy_next_upstream` (next subsection): by default only connection errors and timeouts. An HTTP 500 or 502 from the backend is a valid response and does **not** count unless you list `http_500 http_502 ...`.

**Active health checks** (nginx probes `GET /health` periodically and removes a failing server before any client hits it, with `slow_start` for gentle re-admission) are an NGINX Plus feature (`health_check` directive). Open-source alternatives: put HAProxy or a cloud load balancer in front; use Consul/Kubernetes readiness probes; or the third-party `nginx_upstream_check_module`, which requires recompiling nginx.

**Limitations.** The first failing client requests still pay the cost (up to `proxy_connect_timeout` each) before the server is marked down; with `max_fails=3` three users see a delay. Keep `proxy_connect_timeout` short (2 to 5 s) for LAN backends.

### Retry policy

```nginx
location /api/ {
    proxy_next_upstream error timeout http_502 http_503 http_504;   # when to try another server
    proxy_next_upstream_tries 3;                                    # at most 3 servers in total
    proxy_next_upstream_timeout 10s;                                # total time budget for all attempts
    proxy_pass http://backend_servers/;
}
```

Important safety rule: by default nginx does **not** retry non-idempotent requests (POST, PATCH, LOCK) once the request has been sent to a backend, because the backend may already have performed the action (double payment, double insert). Adding `non_idempotent` to `proxy_next_upstream` removes that protection; do it only if the endpoint is idempotent (idempotency key, PUT semantics). Retried GET and HEAD are generally safe (RFC 9110, safe methods). Without `proxy_next_upstream_tries`, a slow failing endpoint can multiply load by the number of servers.

### Session handling and balancing methods

| Method | Directive | Behaviour | Caution |
| --- | --- | --- | --- |
| Round robin | (default) | in turn, honours `weight` | ignores request cost |
| Least connections | `least_conn;` | fewest active connections | good for uneven request durations |
| IP hash | `ip_hash;` | same client IP, same server | many users behind one NAT hit one server; a server change reshuffles clients |
| Generic hash | `hash $cookie_sessionid consistent;` | same key, same server; `consistent` (ketama) moves few keys when servers change | key must be present on every request |
| URI hash | `hash $request_uri consistent;` | same URL, same server: raises per-server cache hit rate |  |
| Random two | `random two least_conn;` | pick two at random, use the less loaded (nginx >= 1.15.1) | good with several load balancers |
| Sticky cookie | `sticky cookie srv_id ...;` | NGINX Plus only |  |

The robust design is stateless backends: store sessions in Redis or a database, or use signed tokens (JWT), then no stickiness is needed and any server can fail without logging users out.

### Shared upstream state

Without `zone`, every worker process keeps its own copy of the upstream group, so `max_fails` counts and `least_conn` connection counts are per worker. With `zone name size;` they are shared and, in NGINX Plus or with the `resolve` option, can be changed at runtime. Add a zone to every upstream in production (64 KB per few servers is enough).

### DNS resolution and changing container IPs

**Problem.** nginx resolves the host names in `upstream` and `proxy_pass` **once, when the configuration loads**. If a container is recreated and gets a new IP (`docker compose up -d api`), nginx keeps sending to the old address and returns 502 until a reload. If a name cannot be resolved at start, nginx refuses to start: `host not found in upstream "api"`.

**Solutions** (choose one):

```nginx
# 1. reload after changing backends (simple)
#    docker compose exec nginx nginx -s reload

# 2. resolve at request time with a variable in proxy_pass
resolver 127.0.0.11 valid=10s ipv6=off;       # Docker's embedded DNS; use your DNS server elsewhere
server {
    location /api/ {
        set $api_backend http://api:3001;      # variable forces runtime resolution
        rewrite ^/api/(.*)$ /$1 break;          # with a variable, no automatic URI replacement happens: do it yourself
        proxy_pass $api_backend;                # and no URI part on proxy_pass
    }
}

# 3. upstream with the resolve parameter (open source from nginx 1.27.3; earlier NGINX Plus only)
upstream backend_servers {
    zone backend_servers 64k;
    server api:3001 resolve;
}
resolver 127.0.0.11 valid=10s;
```

The variable form also lets nginx start when the backend is not up yet (it fails per request with 502 instead of at boot). Set `valid=` to a short time so that re-created containers are noticed quickly; nginx otherwise honours the DNS TTL.

### Graceful backend removal (draining)

```bash
# 1. mark the server down in the upstream block (or comment it out)
#    server api2:3001 down;
# 2. reload: new requests skip it, in-flight requests continue on old workers
docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload
# 3. wait until its connections drain, then stop and update it
docker compose exec nginx sh -c 'wget -qO- http://127.0.0.1/nginx_status'     # watch active connections (section 17)
docker compose stop api2
```

Long-lived connections (WebSockets, SSE) keep old workers alive. Bound that with `worker_shutdown_timeout 60s;` in the main context; after it nginx closes remaining connections and clients must reconnect. NGINX Plus has a `drain` parameter that does this per server without a reload.

### High availability of nginx itself

One nginx is a single point of failure. Common designs:

| Design | How | Trade-off |
| --- | --- | --- |
| Cloud load balancer in front of 2+ nginx | AWS ALB/NLB, GCP, Azure LB health-check each nginx | simplest; extra cost |
| Floating IP with keepalived (VRRP) | two nginx hosts share one virtual IP; the standby takes it over in seconds | needs L2 network or a provider that allows IP reassignment |
| DNS with several A records | clients pick one | slow failover (TTL), clients may cache a dead address |
| Anycast / managed CDN | provider handles it | vendor dependence |

```conf
# /etc/keepalived/keepalived.conf on the primary (standby: state BACKUP, priority 90)
vrrp_script chk_nginx { script "/usr/bin/pgrep nginx"  interval 2  fall 2  rise 2 }
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    virtual_ipaddress { 192.0.2.10/24 }
    track_script { chk_nginx }
}
```

Keep both nginx configurations identical (Git plus automated deployment, section 18) and test the failover by stopping nginx on the primary.

### Deployment strategies with nginx

```nginx
# Blue-green: switch the pool by editing one include and reloading
# /etc/nginx/conf.d/active-pool.conf :   map $host $pool { default green; }
upstream blue  { server app-blue:3000;  zone blue 64k; }
upstream green { server app-green:3000; zone green 64k; }
location / { proxy_pass http://$pool; }          # variable naming an upstream group is allowed

# Canary: 10 % of clients to the new version, stable per client
split_clients "${remote_addr}${http_user_agent}" $pool {
    10%  canary;
    *    stable;
}
upstream stable { server app-v1:3000; zone stable 64k; }
upstream canary { server app-v2:3000; zone canary 64k; }
location / { proxy_pass http://$pool; }
```

| Strategy | Procedure | Verification | Rollback |
| --- | --- | --- | --- |
| Rolling | for each backend: `down`, reload, update, health check, re-enable | `curl` the new instance directly; watch 5xx rate | re-enable the old build |
| Blue-green | start green, smoke-test it on an internal hostname, flip the pool, reload | compare error rate and latency of the two pools | flip back and reload (seconds) |
| Canary | start with 1 to 10 percent, raise step by step | per-pool `$upstream_status` and `$request_time` in the JSON log | set the canary share to 0 and reload |

### Failure scenarios

| Failure | What the client sees | nginx log | Fix or mitigation |
| --- | --- | --- | --- |
| One backend crashes | brief delay, then success (retry) | `connect() failed (111: Connection refused) while connecting to upstream` | `max_fails`, `proxy_next_upstream`, short connect timeout |
| All backends down | 502, or 503 with `no live upstreams` | `no live upstreams while connecting to upstream` | `backup` servers, static maintenance page via `error_page 502 503` |
| Backend slow | 504 after `proxy_read_timeout` | `upstream timed out (110: Connection timed out) while reading response header` | timeouts, caching with `proxy_cache_use_stale`, capacity |
| DNS outage or changed IP | 502 or startup failure | `host not found in upstream`; `recv() failed (111)` to an old IP | `resolver` plus variable, section above |
| nginx process dies | connection refused | none (process gone) | `restart: unless-stopped`, systemd `Restart=on-failure`, second node |
| Whole host down | timeout | none | HA design above |
| Expired certificate | browser warning; clients fail | none from nginx | renewal monitoring, section 14 |

**Cross-links.** Section 7 (balancing basics), section 12 (timeouts), section 16 (stale cache as a failure buffer), section 17 (finding the cause).

## 16. Caching correctness and performance

Corrections to section 8: being idempotent (a safe `GET`) does not make a response cacheable; what matters is whether the **same bytes may be served to every user**. And the safe default for private data is to bypass the shared cache, not to build a per-user cache key.

### What nginx caches by default

- Methods: `GET` and `HEAD` (`proxy_cache_methods`).
- Status codes: only those listed in `proxy_cache_valid`; with the bare form `proxy_cache_valid 10m;` that is 200, 301 and 302.
- It obeys backend headers: `Cache-Control: private | no-store | no-cache`, `Expires`, `X-Accel-Expires`, and `Vary: *` prevent storing. A response with `Set-Cookie` is not stored by default.
- It does **not** look at the request: a request carrying `Authorization` or a session cookie can still be answered from (or stored into) the shared cache unless you add the rules below.

```nginx
# make backends in charge of freshness: nginx honours Cache-Control/Expires from the app
proxy_cache_valid 200 301 302 10m;       # fallback TTL when the backend sent none
proxy_cache_valid 404 1m;
proxy_cache_valid any 0;                 # do not cache anything else
```

Best practice: have the application send `Cache-Control: public, max-age=60` on cacheable endpoints and `Cache-Control: private, no-store` on personalised ones; nginx then needs little configuration.

### Cache key

The default key is `$scheme$proxy_host$request_uri`. Change it when representations differ by something else:

```nginx
proxy_cache_key "$scheme$request_method$host$request_uri";                  # multi-host server
proxy_cache_key "$scheme$host$uri$is_args$arg_page$arg_lang";               # only these query params matter
proxy_cache_key "$scheme$host$request_uri$http_accept_language";            # language-dependent body
```

If the key omits something that changes the body, one user's version is served to others (cache poisoning/leakage). If it includes noise (tracking parameters such as `utm_source`), the hit rate collapses. Nginx also understands the `Vary` response header, but limits it to a few headers; a safer design is to put the varying input in the key explicitly.

### Bypass versus no-cache

| Directive | Controls | Effect |
| --- | --- | --- |
| `proxy_cache_bypass cond;` | reading | if `cond` is non-empty and not `0`, do not answer from the cache; go to the backend. The fresh response may still be stored |
| `proxy_no_cache cond;` | writing | if `cond` is non-empty and not `0`, do not store the response |

For private traffic use **both**, with the same condition, so the request is neither answered from nor written to the shared cache:

```nginx
map "$http_authorization$cookie_sessionid" $skip_cache {
    default 1;           # anything present -> skip
    ""      0;           # no credentials -> cacheable
}

location /api/ {
    proxy_cache api_cache;
    proxy_cache_valid 200 1m;
    proxy_cache_bypass $skip_cache;
    proxy_no_cache     $skip_cache;
    add_header X-Cache-Status $upstream_cache_status always;
    proxy_pass http://backend_servers/;
}
```

Caution about a pattern seen in common snippets: `proxy_cache_bypass $http_cache_control;` lets any client skip your cache by sending a `Cache-Control` header, which means anyone can send every request to the backend and defeat the cache's load protection. Allow bypass only with a secret you control (next section).

### Invalidation and refresh

Options from simplest to most precise:

1. **Short TTL** (`proxy_cache_valid 200 30s`): accept bounded staleness.
2. **Versioned URLs**: `/api/products?v=1842` or `/assets/app.3f9a1c.js`; new content gets a new key, no invalidation needed.
3. **Forced refresh with a secret header** (open source): the bypassed response replaces the cached one.

```nginx
map $http_x_cache_refresh $refresh_cache {
    "s3cr3t-token" 1;       # keep the value in an include file, not in Git
    default        0;
}
location /api/ {
    proxy_cache api_cache;
    proxy_cache_bypass $refresh_cache;     # still stored afterwards, because proxy_no_cache is not set
    proxy_pass http://backend_servers/;
}
```

```bash
curl -s -o /dev/null -H 'X-Cache-Refresh: s3cr3t-token' https://example.com/api/products     # refresh one URL
```

4. **Purge**: the `proxy_cache_purge` directive is NGINX Plus; the open-source equivalent is the third-party `ngx_cache_purge` module, or deleting files by key hash:

```bash
key="httpsexample.com/api/products"                  # exactly the proxy_cache_key string
hash=$(printf '%s' "$key" | md5sum | cut -d' ' -f1)
# with proxy_cache_path ... levels=1:2 the file is /<path>/<last char>/<2 chars before>/<hash>
rm -f "/data/nginx/cache/${hash: -1}/${hash: -3:2}/$hash"
# add levels=1:2 to proxy_cache_path to use this layout
```

Purging individual files works only with `levels=` set; wiping the whole directory (`rm -rf /data/nginx/cache/*`) is a blunt but valid reset while nginx runs.

### Cache stampede and stale content

**Problem.** When a popular item expires, hundreds of concurrent requests all miss and hit the slow backend (the demo's 7-second endpoint) at once.

```nginx
proxy_cache_lock on;                 # only one request fetches a missing item; others wait
proxy_cache_lock_timeout 5s;
proxy_cache_revalidate on;           # ask the backend with If-Modified-Since/If-None-Match; 304 refreshes cheaply
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;   # serve stale during failures
proxy_cache_background_update on;    # refresh in the background while serving the stale copy
```

`updating` serves the stale copy while a refresh is in progress; `proxy_cache_background_update` performs that refresh asynchronously. nginx also honours `stale-while-revalidate` and `stale-if-error` extensions from the backend (RFC 5861). Trade-off: users may see content up to `TTL + stale window` old, in exchange for availability when the backend fails: a useful complement to section 15.

### Observing the cache

`$upstream_cache_status` values: `MISS`, `HIT`, `EXPIRED` (stale, being refreshed), `STALE` (served stale), `UPDATING`, `REVALIDATED`, `BYPASS`.

```nginx
log_format cache '$remote_addr [$time_local] "$request" $status $upstream_cache_status $request_time';
access_log /var/log/nginx/cache.log cache;
```

```bash
# hit ratio per status
awk '{print $(NF-1)}' /var/log/nginx/cache.log | sort | uniq -c | sort -rn
# JSON log variant (section 17)
jq -r '.cache_status' /var/log/nginx/access_json.log | sort | uniq -c

# manual check: the second request must say HIT
curl -sI https://example.com/api/process-data | grep -i x-cache-status
curl -sI https://example.com/api/process-data | grep -i x-cache-status
```

A falling hit ratio after a deploy usually means the key changed, a new `Set-Cookie` header appeared, or the backend started sending `Cache-Control: no-cache`. Check disk with `du -sh /data/nginx/cache`; `max_size` evicts the least recently used items, and `inactive` removes unused ones.

### Compression correctness

```nginx
gzip on;
gzip_vary on;                 # adds Vary: Accept-Encoding so caches keep compressed and plain variants apart
gzip_proxied any;             # compress responses to proxied requests (default: off for requests with Via)
gzip_min_length 1000;
gzip_comp_level 5;
gzip_types text/css application/json application/javascript text/xml application/xml image/svg+xml;

# precompressed static files: build once, serve at zero CPU cost (module http_gzip_static)
gzip_static on;               # app.js is answered from app.js.gz when the client sends Accept-Encoding: gzip
```

```bash
find /var/www/site/assets -type f \( -name '*.js' -o -name '*.css' -o -name '*.svg' \) -exec gzip -k -9 {} \;
curl -sI -H 'Accept-Encoding: gzip' https://example.com/assets/app.js | grep -iE 'content-encoding|vary'
```

Security note: compressing responses that mix a secret (CSRF token, session data) with attacker-controlled text enables the BREACH attack (Gluck, Harris, Prado, 2013), because the compressed length leaks information. Mitigations: do not compress such pages, randomize or mask secrets per response, rate limit, and use SameSite cookies. Static files and public JSON are safe to compress. Brotli needs the third-party `ngx_brotli` module (`brotli on; brotli_comp_level 5;`) and is typically used for static precompressed assets.

### Static-file performance

```nginx
http {
    sendfile on;                     # kernel copies file to socket, no user-space copy
    tcp_nopush on;                   # with sendfile: send headers and file start in full packets
    tcp_nodelay on;                  # keep-alive connections: send small packets at once
    aio threads;                     # (optional, nginx built with threads) avoid blocking on slow disks

    open_file_cache          max=2000 inactive=20s;   # cache file descriptors and metadata
    open_file_cache_valid    30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;
}
```

These settings help when serving many small static files from disk; they do nothing for proxied dynamic traffic. Measure before and after (below); on a small VPS the defaults are often adequate. `sendfile` can misbehave on some network or virtualised file systems (VirtualBox shared folders): serve stale content if you edit files there; set `sendfile off;` in that dev environment.

### HTTP/2 and HTTP/3

|  | HTTP/1.1 | HTTP/2 (RFC 9113) | HTTP/3 (RFC 9114 over QUIC, RFC 9000) |
| --- | --- | --- | --- |
| Transport | TCP | TCP + TLS | UDP (QUIC with built-in TLS 1.3) |
| Multiplexing | one request per connection | many streams per connection; head-of-line blocking at TCP level | independent streams; no TCP head-of-line blocking |
| Header compression | none | HPACK | QPACK |
| nginx requirement | default | built with `--with-http_v2_module` | built with `--with-http_v3_module` (nginx >= 1.25.0) |

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;                                  # nginx >= 1.25.1; older: listen 443 ssl http2;

    listen 443 quic reuseport;                 # HTTP/3 over UDP; reuseport once per address:port in the whole config
    listen [::]:443 quic reuseport;
    http3 on;

    server_name example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;             # QUIC needs TLS 1.3

    add_header Alt-Svc 'h3=":443"; ma=86400' always;   # tells browsers they may switch to HTTP/3
}
```

Open UDP 443 in the firewall and in Docker (`ports: ["443:443/tcp", "443:443/udp"]`). Test:

```bash
curl -sI --http2 https://example.com | head -1          # HTTP/2 200
curl -sI --http3 https://example.com | head -1          # needs a curl built with HTTP/3
nginx -V 2>&1 | grep -o 'with-http_v[23]_module'
```

Browsers first connect over TCP, read `Alt-Svc`, then try QUIC; if UDP is blocked they silently fall back to HTTP/2, so HTTP/3 is safe to enable.

### Performance measurement

Measure latency percentiles (p95, p99), not only averages, and watch errors and resource use at the same time.

```bash
# wrk: 4 threads, 100 connections, 30 s, latency distribution
wrk -t4 -c100 -d30s --latency https://example.com/api/products

# hey: 2000 requests, 50 concurrent
hey -n 2000 -c 50 https://example.com/api/products

# k6: scripted scenario with thresholds
cat > load.js <<'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = { vus: 50, duration: '30s', thresholds: { http_req_duration: ['p(95)<300'], http_req_failed: ['rate<0.01'] } };
export default function () {
  const r = http.get('https://example.com/api/products');
  check(r, { 'status 200': (x) => x.status === 200 });
  sleep(1);
}
EOF
k6 run load.js

# while the test runs, on the server
docker stats --no-stream; ss -s; tail -f /var/log/nginx/error.log
```

Compare cache MISS and HIT runs, gzip on and off, and keepalive on and off, one change at a time. Test from a different machine than the server, and never against a production system without permission; load tests also trigger your own rate limits (section 9), so exempt the tester IP or expect 429 responses.

**Cross-links.** Sections 8, 11 (browser caching), 15 (stale responses), 17 (logs for cache status), RFC 9111, RFC 5861, RFC 9113, RFC 9114.

## 17. Logging, monitoring and troubleshooting

### Access and error logs

| Log | Default path (package) | Docker image | Content |
| --- | --- | --- | --- |
| Access log | `/var/log/nginx/access.log` | symlink to stdout (`docker logs`) | one line per request |
| Error log | `/var/log/nginx/error.log` | symlink to stderr | problems, upstream failures, config warnings |

```nginx
error_log /var/log/nginx/error.log warn;      # levels: debug info notice warn error crit alert emerg
access_log /var/log/nginx/access.log combined;
access_log off;                               # inside a location, e.g. health checks or static assets
```

```bash
sudo tail -f /var/log/nginx/error.log
docker compose logs -f --tail=100 nginx
docker logs --since 10m <nginx-container> 2>&1 | grep -E ' (4|5)[0-9]{2} '
```

In a container, `access_log /var/log/nginx/x.log` writes inside the container's writable layer and is lost on recreation; send to stdout (`/dev/stdout`) or mount a volume.

### Structured JSON logs with request IDs and timing

```nginx
log_format json_analytics escape=json '{'
    '"time":"$time_iso8601",'
    '"request_id":"$request_id",'
    '"remote_addr":"$remote_addr",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'                                   # path without query string: avoids leaking tokens
    '"status":$status,'
    '"bytes":$body_bytes_sent,'
    '"request_time":$request_time,'                    # total, including sending to a slow client
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_connect_time":"$upstream_connect_time",'
    '"upstream_header_time":"$upstream_header_time",'  # time to first byte from the backend
    '"upstream_response_time":"$upstream_response_time",'
    '"cache_status":"$upstream_cache_status",'
    '"user_agent":"$http_user_agent"'
'}';

access_log /var/log/nginx/access_json.log json_analytics;
# pass the same ID to the backend and back to the client to follow one request across services
proxy_set_header X-Request-ID $request_id;
add_header X-Request-ID $request_id always;
```

Reading the timings: `request_time` minus `upstream_response_time` is time spent buffering to the client; a large `upstream_connect_time` points at network or backend accept queues; a large `upstream_header_time` with a small connect time means the application is slow. With several retries these variables hold comma-separated lists, one per attempt.

```bash
# 10 slowest requests, error rate, requests per status
jq -r '[.request_time,.uri,.status]|@tsv' /var/log/nginx/access_json.log | sort -rn | head
jq -r .status /var/log/nginx/access_json.log | sort | uniq -c | sort -rn
jq -r 'select(.status>=500)|[.time,.uri,.upstream_addr,.upstream_status]|@tsv' /var/log/nginx/access_json.log | tail
```

Ship the file to Loki/Promtail, Filebeat/Elasticsearch, or Vector; JSON needs no parsing rules.

### Rotation and retention

```conf
# /etc/logrotate.d/nginx  (installed by the Debian package; adjust)
/var/log/nginx/*.log {
    daily
    rotate 14                 # retention: 14 days (also your privacy retention period)
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)    # reopen log files
    endscript
}
```

Without `USR1`, nginx keeps writing to the renamed (and eventually deleted) file and the disk fills silently (`df -h`, `lsof +L1 | grep nginx`). In Docker, limit the json-file driver instead: `logging: { driver: json-file, options: { max-size: "10m", max-file: "5" } }`.

### Basic metrics with stub\_status

```nginx
server {
    listen 127.0.0.1:8081;                  # not public
    location = /nginx_status {
        stub_status;                          # needs --with-http_stub_status_module
        access_log off;
        allow 127.0.0.1;
        allow 172.16.0.0/12;                  # Docker network, if the exporter runs there
        deny all;
    }
}
```

```text
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

`accepts` equals `handled` unless a resource limit rejected connections; `requests / handled` is requests per connection (keepalive reuse); `Waiting` is idle keepalive connections; a growing `Writing` means slow clients or backends. Export to Prometheus:

```yaml
  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest      # pin a version
    command: ["--nginx.scrape-uri=http://nginx:8081/nginx_status"]
    ports: ["127.0.0.1:9113:9113"]
```

### Dashboards and alerts

| Signal | Source | Alert when |
| --- | --- | --- |
| Error rate | 5xx / all requests from logs or `nginx_http_requests_total` | above 1 percent for 5 minutes |
| Latency | p95 of `request_time`, `upstream_response_time` | p95 above your SLO |
| Traffic | requests per second | sudden drop to 0 or tenfold spike |
| Saturation | active connections, worker CPU, open files, disk of cache and logs | above 80 percent of limits |
| 429 / 499 counts | logs | spikes (abuse, or clients giving up) |
| Certificate expiry | script in section 14 | less than 14 days |
| Upstream health | `upstream_status` of failures, `no live upstreams` lines | any occurrence |

### Layered health checks

"nginx is up" and "the application works" are different questions:

```nginx
location = /healthz {                  # liveness: nginx answers by itself
    access_log off;
    default_type text/plain;
    return 200 "ok\n";
}
location = /readyz {                   # readiness: goes through to the app and its dependencies
    access_log off;
    proxy_pass http://backend_servers/health;
    proxy_connect_timeout 2s;
    proxy_read_timeout 3s;
}
```

```yaml
  nginx:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1/healthz"]
      interval: 15s
      timeout: 3s
      retries: 3
```

Point cloud load balancers at `/readyz`, orchestrators' liveness probes at `/healthz`.

### HTTP error diagnosis

| Status | Meaning in nginx | First checks |
| --- | --- | --- |
| 400 | malformed request, oversized header, mTLS missing certificate | `large_client_header_buffers`; client certificate; error log line |
| 403 | permission denied, `deny`, no index file, bad client certificate | `namei -l path`, `allow/deny`, `error.log` text (section 11) |
| 404 | no file or backend route | wrong `root`/`alias`, trailing slash in `proxy_pass` (section 4), location precedence (section 11) |
| 413 | body larger than `client_max_body_size` (default 1m) | raise it in the right location; check the backend's own limit |
| 429 | `limit_req`/`limit_conn` rejection (when `limit_req_status 429`) | tune `rate`, `burst`; real IP behind proxies (section 12) |
| 499 | nginx-specific: the client closed the connection before the response | slow backend, impatient client or mobile network; compare with `request_time` |
| 500 | backend error, or nginx internal redirection cycle | app logs; `rewrite or internal redirection cycle` |
| 502 | invalid or no response from the backend | backend down or refused, wrong port, crashed worker, DNS/IP change, keepalive race, upstream TLS error |
| 503 | no live upstreams, or rate limit default | `no live upstreams`; `limiting requests` line; maintenance flag |
| 504 | backend slower than `proxy_read_timeout` | app profiling; raise timeout only for known long endpoints |

### Network and service diagnostics

```bash
# What does the client see?
curl -v https://example.com/api/users                  # request/response headers, TLS handshake
curl -sS -o /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total} code=%{http_code}\n' https://example.com/
curl --resolve example.com:443:203.0.113.5 https://example.com/     # bypass DNS, target one server

# DNS and sockets
dig +short example.com A; dig +short example.com AAAA; dig +trace example.com
ss -tlnp | grep -E ':(80|443)\b'                        # who listens on 80/443?
sudo lsof -i :80

# TLS
openssl s_client -connect example.com:443 -servername example.com -brief </dev/null

# Services and logs
sudo systemctl status nginx --no-pager; sudo journalctl -u nginx -n 100 --no-pager
sudo nginx -t; sudo nginx -T | less
docker compose ps; docker compose logs --tail=100 nginx api
docker compose exec nginx sh -c 'wget -qO- http://api:3001/health'     # can nginx reach the backend from inside its network?
```

Common start-up errors: `bind() to 0.0.0.0:80 failed (98: Address already in use)` (another nginx, Apache, or a container publishes port 80: find it with `ss -tlnp`), `cannot load certificate ... No such file` (renewal or path), `host not found in upstream` (section 15).

### Debug logging

Needs a binary built with `--with-debug` (check `nginx -V`; Debian ships `nginx-debug`). It is very verbose and slows the server, so restrict it to one client:

```nginx
error_log /var/log/nginx/debug.log debug;     # global debug: do not leave on in production

events {
    debug_connection 203.0.113.5;             # debug only connections from this IP (or CIDR)
}
```

Reload, reproduce one request, read the rewrite and location decisions in `debug.log` ("test location", "using configuration", "rewritten data"), then remove the directives and reload.

### Troubleshooting ladder

Work from the outside in, stop at the first layer that fails:

1. **DNS**: `dig +short host` returns the expected IP (and AAAA). Wrong IP: fix records, wait for TTL.
2. **Network and firewall**: `nc -zv host 443` or `curl -v`. Timeout: security group, `ufw status`, cloud firewall, nginx not listening (`ss -tlnp`).
3. **TLS**: `openssl s_client -servername host`. Certificate expired or wrong name: section 14.
4. **Server and location selection**: `curl -v --resolve ...` plus `nginx -T`; add a temporary `add_header X-Debug-Loc "x" always;` to see which block served. Unexpected default site: section 11.
5. **Access phase**: 401, 403, 429 (auth, allow/deny, limits).
6. **Upstream**: `error.log` lines `connect() failed`, `upstream timed out`, `prematurely closed`; test the backend directly from the nginx network.
7. **Application**: app logs with the same `X-Request-ID`; reproduce with `curl` against the backend port.

**Cross-links.** Section 12 (timeouts, headers), section 14 (certificates), section 15 (upstream failures), section 16 (cache status).

## 18. Docker and production operations

### Network isolation

Only nginx should publish ports. Put the backends on a private network that has no route from outside, and bind admin tools to loopback.

```yaml
services:
  nginx:
    image: nginx:1.28-alpine
    ports: ["80:80", "443:443"]            # the only public ports
    networks: [edge, internal]
  api:
    build: ./backend
    expose: ["3001"]                        # reachable by name inside the network; NOT published to the host
    networks: [internal]
  mongo:
    image: mongo:7
    networks: [internal]
  mongo-express:
    image: mongo-express
    ports: ["127.0.0.1:8081:8081"]          # admin UI only on the host's loopback (reach it via SSH tunnel)
    networks: [internal]

networks:
  edge: {}
  internal:
    internal: true                          # no outbound internet; remove if the backends need it
```

This removes the "backend ports are also open" weakness noted in section 4: `curl http://SERVER_IP:3001` must fail from outside. Docker also edits iptables itself, so `ufw` rules do not block published ports; published ports are exposed regardless. Verify with `nmap -p- SERVER_IP` from another machine.

### Start-up order, readiness and unresolvable upstreams

`depends_on` alone only orders container creation. nginx fails to start when an upstream host name does not resolve, and sends 502 while a backend is still booting. Combine health checks with `condition: service_healthy`:

```yaml
  api:
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3001/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
  nginx:
    depends_on:
      api:
        condition: service_healthy
```

If a backend is recreated and its IP changes, nginx must re-resolve (section 15: `resolver` plus variable, or reload).

### Reproducible images and updates

- Pin the image tag (`nginx:1.28.0-alpine`, example version: check the current stable release), not `latest`. Pin a digest (`nginx@sha256:...`) for fully reproducible builds.
- Update deliberately: read release notes (security advisories on nginx.org), build the new image, run `nginx -t` with your real config, deploy to staging, run the tests below, then production.
- Scan images: `docker scout cves nginx:1.28-alpine` or `trivy image nginx:1.28-alpine`.
- Zero-downtime nginx binary upgrade on a host install uses the `USR2` + `QUIT` signal sequence; in Docker you recreate the container, which drops connections briefly unless a second nginx or a load balancer is in front.

### Non-root, read-only and permissions

The official image starts the master as root (to bind port 80), workers as `nginx`. To run fully unprivileged use `nginxinc/nginx-unprivileged` (listens on 8080 and 8443) and a read-only file system with explicit writable paths:

```yaml
  nginx:
    image: nginxinc/nginx-unprivileged:1.28-alpine
    ports: ["80:8080"]                       # host 80 -> container 8080
    read_only: true
    tmpfs:
      - /tmp
      - /var/cache/nginx                      # proxy_cache_path outside this dir needs its own volume
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - nginx_cache:/data/nginx/cache         # writable named volume owned by the nginx user
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
```

Update `listen` to 8080 in the config when using the unprivileged image. A permission error like `mkdir() "/data/nginx/cache" failed (13: Permission denied)` means the volume is owned by root: fix with `chown` in an init container or `docker run --rm -v nginx_cache:/d alpine chown 101:101 /d` (the `nginx` user's UID in the official images is 101).

### Configuration templates with environment variables

The official image renders every `/etc/nginx/templates/*.template` file with `envsubst` into `/etc/nginx/conf.d/` at start. Nginx variables such as `$host` would be mangled by a naive `envsubst`; the image only substitutes variables that exist in the environment, and `NGINX_ENVSUBST_FILTER` narrows it further:

```yaml
  nginx:
    image: nginx:1.28-alpine
    environment:
      SERVER_DOMAIN: example.com
      API_UPSTREAM: api:3001
      NGINX_ENVSUBST_FILTER: "^(SERVER|API)_"       # only these variable names are substituted
    volumes:
      - ./nginx/default.conf.template:/etc/nginx/templates/default.conf.template:ro
```

```nginx
# nginx/default.conf.template
server {
    listen 80;
    server_name ${SERVER_DOMAIN} www.${SERVER_DOMAIN};
    location /api/ { proxy_pass http://${API_UPSTREAM}/; proxy_set_header Host $host; }   # $host stays untouched
}
```

Check the rendered result: `docker compose exec nginx cat /etc/nginx/conf.d/default.conf`.

### Safe deployment workflow (replaces `docker compose down` deployments)

The tutorials stop the whole stack (`down`, then `up`), which drops every connection and causes downtime. A safer sequence:

```bash
git pull --ff-only
docker compose config -q                                   # compose file valid?
docker compose build api                                   # build before touching anything
# 1. validate the candidate nginx config in a throw-away container on the real network
docker compose run --rm --no-deps --entrypoint nginx nginx -t
# 2. update only what changed, keep the others running
docker compose up -d --no-deps api
# 3. apply nginx config without dropping connections
docker compose exec nginx nginx -s reload
# 4. smoke test, and roll back if it fails
./tests/smoke.sh https://example.com || { git checkout HEAD~1 -- nginx/ && docker compose exec nginx nginx -s reload; echo ROLLED BACK; }
```

Rollback needs a retained last-known-good config: keep it in Git, tag every deployed revision (`git tag deploy-$(date +%F-%H%M)`), and keep the previous image tag available.

### Automated configuration tests

```bash
#!/bin/sh
# tests/smoke.sh <domain> : routing, redirects, headers, auth, cache, compression
D=${1:?domain, e.g. example.com}
B=https://$D
fail=0
code()  { curl -s -o /dev/null -w '%{http_code}' "$@"; }
check() { if [ "$2" = "$3" ]; then echo "ok   $1"; else echo "FAIL $1 (got $2, want $3)"; fail=1; fi; }

check "http redirects to https"   "$(code http://$D/)" 301
check "homepage"                  "$(code $B/)" 200
check "api reachable"             "$(code $B/api/users)" 200
check "hsts header"               "$(curl -sI $B/ | grep -ci '^strict-transport-security')" 1
check "server version hidden"     "$(curl -sI $B/ | grep -i '^server:' | grep -cE '[0-9]+\.[0-9]+')" 0
check "admin needs auth"          "$(code $B/mongo/)" 401
check ".env blocked"              "$(code $B/.env)" 403
check "cache hit on 2nd request"  "$(curl -s -o /dev/null $B/api/process-data; curl -sI $B/api/process-data | grep -ci 'x-cache-status: hit')" 1
check "gzip enabled"              "$(curl -sI -H 'Accept-Encoding: gzip' $B/api/big-data | grep -ci 'content-encoding: gzip')" 1
check "direct IP rejected"        "$(code -k --max-time 5 https://$(dig +short $D | head -1)/)" 000
exit $fail
```

Adjust paths and expected codes to your site (the 301 check uses the plain-HTTP URL). Run the script in CI after deployment to staging and from cron against production.

### Resource limits and system limits

```yaml
  nginx:
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 256M }
    ulimits:
      nofile: { soft: 65535, hard: 65535 }     # matches worker_rlimit_nofile (section 10)
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "5" }
```

Bound the proxy cache (`max_size`), the upload temp files (`df -h` alerts), and the number of connections (`worker_connections`). Without limits one noisy tenant can fill the disk and stop every site.

### Backup and recovery

What you need to rebuild nginx on a replacement host in minutes:

| Item | Where | Backup method |
| --- | --- | --- |
| Configuration | Git repository (`nginx/`, `docker-compose.yml`) | Git remote |
| Certificates, account key, renewal config | `/etc/letsencrypt` | encrypted archive, off-host |
| Secrets (`.htpasswd`, client CAs, API tokens) | secrets store or encrypted file | SOPS/age, password manager, Docker secrets |
| DNS records | DNS provider | zone export, infrastructure as code |

```bash
sudo tar czf - /etc/letsencrypt | age -r <public-key> > letsencrypt-$(date +%F).tar.gz.age       # encrypt before upload
# restore on a new host: install docker, git clone, restore /etc/letsencrypt, point DNS (or float the IP), then
docker compose up -d && docker compose exec nginx nginx -t
```

Rehearse the restore on a scratch VM; an untested backup is a hope. Because certificates can be re-issued, losing `/etc/letsencrypt` costs a rate-limited re-issue rather than an outage, provided DNS and port 80 work.

### Graceful shutdown

nginx stops gracefully on `SIGQUIT` (finishes in-flight requests), immediately on `SIGTERM`. The official image sets `STOPSIGNAL SIGQUIT`, so `docker stop` is graceful, but Docker kills the container after 10 s by default:

```yaml
  nginx:
    stop_grace_period: 60s       # >= worker_shutdown_timeout
```

```nginx
worker_shutdown_timeout 50s;     # main context: cut off connections that outlive the grace period
```

### Configuration as code

```yaml
# .github/workflows/nginx.yml : validate every pull request
name: nginx-config
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: syntax check
        run: |
          docker run --rm -v $PWD/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro \
            nginx:1.28-alpine sh -c 'sed -i "s/server \(api[0-9]*\):/server 127.0.0.1:/" /etc/nginx/conf.d/default.conf; nginx -t'
      - name: static analysis (gixy)
        run: pipx install gixy && gixy nginx/default.conf
```

(The `sed` line only replaces service names with a resolvable address because a lone container cannot resolve `api`; adapt it to your naming, or test inside the Compose network instead.) `gixy` flags common mistakes: unsafe alias, host header spoofing, missing `proxy_set_header` inheritance. For servers outside Docker use Ansible:

```yaml
- name: deploy nginx site
  hosts: web
  become: true
  tasks:
    - ansible.builtin.template:
        src: templates/site.conf.j2
        dest: /etc/nginx/sites-available/example.conf
      notify: [validate nginx, reload nginx]      # handlers run in this order, only if the file changed
  handlers:
    - name: validate nginx
      ansible.builtin.command: nginx -t
      changed_when: false
    - name: reload nginx
      ansible.builtin.service: { name: nginx, state: reloaded }
```

The handler runs only when the file changed, and `reloaded` keeps connections; if validation fails the play stops before the reload, but the bad file is already on disk, so fix it before the next restart.

**Cross-links.** Sections 5 (Compose basics), 14 (certificate renewal), 15 (DNS, deployments), 17 (logs and health checks).

## 19. Advanced topics and labs

### TCP and UDP proxying (stream context)

The `stream` context proxies raw TCP/UDP (databases, MQTT, DNS, SSH) with no HTTP awareness. It sits next to `http`, not inside it, and needs `--with-stream` (or the dynamic stream module). Its logs, variables and directives differ from HTTP (`proxy_pass` has no URI part, no `location`).

```nginx
stream {
    upstream pg {
        server 10.0.0.5:5432 max_fails=2 fail_timeout=10s;
        server 10.0.0.6:5432 backup;
    }
    server {
        listen 5432;
        proxy_pass pg;
        proxy_connect_timeout 3s;
        proxy_timeout 10m;                 # idle time for the whole connection
    }

    upstream dns { server 10.0.0.53:53; server 10.0.0.54:53; }
    server {
        listen 53 udp reuseport;           # UDP
        proxy_pass dns;
        proxy_responses 1;                 # one reply per query
    }

    log_format basic '$remote_addr [$time_local] $protocol $status $bytes_sent $bytes_received $session_time';
    access_log /var/log/nginx/stream.log basic;
}
```

Expose a database to the internet only with TLS (`listen 5433 ssl;` with `ssl_certificate`) plus IP allow-lists (`allow`/`deny` work in `stream`), and prefer a VPN. Test: `psql -h proxy-host -p 5432 -U app db`, `dig @proxy-host -p 53 example.com`.

### TLS passthrough and SNI routing

With `ssl_preread` nginx reads the SNI of the TLS ClientHello without decrypting and routes the encrypted stream to the right backend. The backends keep their own certificates; nginx cannot inspect or cache the HTTP inside.

```nginx
stream {
    map $ssl_preread_server_name $backend {
        app.example.com    app_pool;
        api.example.com    api_pool;
        default            app_pool;
    }
    upstream app_pool { server 10.0.1.10:443; }
    upstream api_pool { server 10.0.1.20:443; }

    server {
        listen 443;
        ssl_preread on;
        proxy_pass $backend;
    }
}
```

```bash
curl --resolve app.example.com:443:PROXY_IP https://app.example.com/ -v    # certificate presented must be the app backend's
```

### PROXY protocol

A TCP proxy (HAProxy, AWS NLB) hides the client IP from the next hop because there is no HTTP header to add. The PROXY protocol prepends a small header carrying the original addresses. Both sides must agree; a mismatch breaks every connection.

```nginx
# nginx behind a load balancer that sends PROXY protocol
server {
    listen 443 ssl proxy_protocol;
    set_real_ip_from 10.0.0.0/8;          # the load balancer
    real_ip_header proxy_protocol;        # $remote_addr becomes the real client
    proxy_set_header X-Forwarded-For $proxy_protocol_addr;
}

# nginx stream proxy that forwards the client address to a backend which expects PROXY protocol
stream { server { listen 5432; proxy_pass pg; proxy_protocol on; } }
```

### Kubernetes

In Kubernetes the proxy role is played by an Ingress controller or a Gateway implementation: Services give stable virtual IPs and DNS names to pods, an Ingress or Gateway API route maps host and path to a Service, and a controller turns that into proxy configuration.

Status warning (verified October 2026 against the Kubernetes project announcements): the community controller **kubernetes/ingress-nginx** was retired in March 2026; upstream releases, bug fixes and security patches ended, and the Kubernetes Steering and Security Response Committees urge migration to the Gateway API or another controller. The commercial/F5 products (NGINX Ingress Controller from F5, NGINX Gateway Fabric) are separate projects. The tutorial's statement that the NGINX Ingress controller is among the most popular is therefore historical; the concepts (host/path routing, TLS termination, load balancing inside the cluster, a cloud load balancer as the public entry point) still apply to whichever controller you choose.

```bash
kubectl get pods --all-namespaces --selector app.kubernetes.io/name=ingress-nginx   # do you depend on the retired controller?
```

```yaml
# Gateway API route: the modern replacement for an Ingress rule
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: shop }
spec:
  parentRefs: [{ name: public-gateway }]
  hostnames: ["shop.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /api } }]
      backendRefs: [{ name: api, port: 3001 }]
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs: [{ name: web, port: 3000 }]
```

### njs (nginx JavaScript)

njs lets you add small pieces of logic (header manipulation, token checks, custom responses) without a separate service. Keep heavy logic in the application: njs runs inside the worker's event loop.

```nginx
load_module modules/ngx_http_js_module.so;      # main context
http {
    js_import main from /etc/nginx/hello.js;
    server {
        listen 8080;
        location /hello { js_content main.hello; }
    }
}
```

```javascript
// /etc/nginx/hello.js
function hello(r) { r.return(200, `Hello ${r.args.name || 'world'} from njs\n`); }
export default { hello };
```

Test: `curl 'http://localhost:8080/hello?name=Pouria'`.

### Large downloads, ranges and slicing

Browsers and players use `Range` requests (RFC 9110) to seek in video and resume downloads. For large files behind a cache, slice them so that a request for the middle of a 2 GB file does not fetch and store all of it:

```nginx
location /media/ {
    slice 1m;                                         # needs --with-http_slice_module
    proxy_cache media_cache;
    proxy_cache_key $uri$is_args$args$slice_range;
    proxy_set_header Range $slice_range;
    proxy_cache_valid 200 206 1h;
    proxy_pass http://storage:9000;
}
```

```bash
curl -sI -H 'Range: bytes=0-99' https://example.com/media/movie.mp4 | head -3      # expect 206 Partial Content
```

Nginx can also proxy mail protocols (`mail` context, IMAP/POP3/SMTP with an authentication endpoint); that is outside the web scope of these notes.

### Labs with expected results

Use the Compose stack from sections 5 to 9 on a test machine. Each lab is: set up, break or change, observe, fix.

**Lab 1: fix a routing bug.** Goal: `/static/logo.png` must be served from `/var/www/static`, but a regex block steals it.

```nginx
location /static/ { root /var/www; }                   # prefix
location ~* \.(png|jpg)$ { proxy_pass http://app:3000; }  # regex wins over the plain prefix -> wrong block
```

```bash
curl -sI http://localhost/static/logo.png | head -1     # observe the backend's answer (for example 404 from the app)
# fix: location ^~ /static/ { root /var/www; }
nginx -t && nginx -s reload
curl -sI http://localhost/static/logo.png | head -1     # expected: HTTP/1.1 200 OK, Content-Type: image/png
```

**Lab 2: diagnose a 502.**

```bash
docker compose stop api
curl -i http://localhost/api/users                      # expected: 502 Bad Gateway
docker compose exec nginx tail -n 3 /var/log/nginx/error.log
# expected: connect() failed (111: Connection refused) while connecting to upstream ... upstream: "http://172.18.0.4:3001/users"
docker compose start api                                # expected: 200 within a few seconds
# variation: change the port to 3002 in proxy_pass; the same symptom has a different cause, found by the upstream address in the log line
```

**Lab 3: handle a large upload.**

```bash
dd if=/dev/zero of=/tmp/5m.bin bs=1M count=5
curl -s -o /dev/null -w '%{http_code}\n' -F file=@/tmp/5m.bin http://localhost/upload/     # expected: 413
# add client_max_body_size 10m; to location /upload/, reload
curl -s -o /dev/null -w '%{http_code}\n' -F file=@/tmp/5m.bin http://localhost/upload/     # expected: 200
docker compose exec nginx tail -n 2 /var/log/nginx/error.log     # before the fix: client intended to send too large body: 5242880 bytes
```

**Lab 4: proxy a WebSocket.**

```bash
mkdir ws-lab && cd ws-lab && npm init -y >/dev/null && npm i ws >/dev/null
cat > server.js <<'EOF'
const { WebSocketServer } = require('ws');
const wss = new WebSocketServer({ port: 3000 });
wss.on('connection', (s) => s.on('message', (m) => s.send('echo: ' + m)));
EOF
node server.js &
# nginx location /ws/ { proxy_pass http://127.0.0.1:3000; }          # first WITHOUT upgrade headers
npx wscat -c ws://localhost/ws/                                       # expected: error: Unexpected server response: 400 (or 426)
# add proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection $connection_upgrade; reload
npx wscat -c ws://localhost/ws/                                       # expected: Connected; type hi -> echo: hi
sudo nginx -s reload                                                  # existing socket stays open until the client closes; new ones use the new workers
```

**Lab 5: test certificate renewal.**

```bash
docker compose run --rm --entrypoint certbot certbot renew --dry-run
# expected last line: Congratulations, all simulated renewals succeeded
curl -s -o /dev/null -w '%{http_code}\n' http://example.com/.well-known/acme-challenge/x      # expected: 404 (nginx reachable, no such token), not a redirect to HTTPS
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -enddate
```

If the dry run fails with `Connection refused` or `unauthorized`, port 80 or the `/.well-known/acme-challenge/` location is wrong; read `/var/log/letsencrypt/letsencrypt.log`.

**Lab 6: prove private responses are not cached.** Backend route returning per-user data:

```javascript
app.get('/api/me', (req, res) => {      // no Cache-Control sent: the worst case for a shared cache
  const who = (req.get('Cookie') || '').match(/sessionid=(\w+)/)?.[1] || 'anonymous';
  res.json({ user: who, at: Date.now() });
});
```

```bash
# 1. WITHOUT skip rules (bug): alice primes the cache, bob receives alice's data
curl -s -H 'Cookie: sessionid=alice' http://localhost/api/me        # {"user":"alice",...}
curl -s -H 'Cookie: sessionid=bob'   http://localhost/api/me        # BUG: still {"user":"alice",...}
# 2. add the map + proxy_cache_bypass + proxy_no_cache from section 16, reload, clear the cache
docker compose exec nginx sh -c 'rm -rf /data/nginx/cache/*' && docker compose exec nginx nginx -s reload
curl -s -D- -H 'Cookie: sessionid=alice' http://localhost/api/me | grep -iE 'x-cache-status|user'     # BYPASS, alice
curl -s -D- -H 'Cookie: sessionid=bob'   http://localhost/api/me | grep -iE 'x-cache-status|user'     # BYPASS, bob
curl -s -D- http://localhost/api/me | grep -i x-cache-status                                          # MISS (anonymous)
curl -s -D- http://localhost/api/me | grep -i x-cache-status                                          # HIT
```

Expected: personalised requests are always BYPASS and return their own data; only anonymous traffic is cached.

**Lab 7: stop a backend under load.**

```bash
( for i in $(seq 1 200); do curl -s -o /dev/null -w '%{http_code}\n' http://localhost/api/users; sleep 0.1; done | sort | uniq -c ) &
sleep 5; docker compose stop api2
wait
# expected with max_fails + proxy_next_upstream (sections 7 and 15): 200 lines of 200, perhaps a few slower responses; without retry: some 502
docker compose start api2
```

**Lab 8: roll back a bad deployment.**

```bash
git tag deploy-good
# (a) syntax error: remove a semicolon
docker compose exec nginx nginx -t           # expected: [emerg] ... unexpected "}" ; the reload is never run, the running config is unchanged
# (b) valid but wrong: proxy_pass to port 3999
docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload
./tests/smoke.sh example.com                  # expected: FAIL api reachable (got 502, want 200)
git checkout deploy-good -- nginx/ && docker compose exec nginx nginx -s reload
./tests/smoke.sh example.com                  # expected: all ok
```

Write down what you saw in the logs for each lab; the error text patterns from section 17 are what you will search for in a real incident.

**Cross-links.** Sections 10 to 18 for the theory behind each lab.

## 20. Cheat sheet, exam-style questions and answer key

### Commands

| Task | Command |
| --- | --- |
| Version / build info | `nginx -v`, `nginx -V` |
| Syntax check | `sudo nginx -t` |
| Reload / quit | `nginx -s reload`, `nginx -s quit` (or `sudo systemctl reload nginx`) |
| Enable a site | `sudo ln -s /etc/nginx/sites-available/X /etc/nginx/sites-enabled/X` |
| Disable a site | `sudo rm /etc/nginx/sites-enabled/X` |
| Check inside Docker | `docker exec -it <nginx> nginx -t` |
| Run the stack | `docker compose up --build -d`, `docker compose down` |
| Self-signed cert | `openssl req -x509 -newkey rsa:2048 -nodes -days 365 -keyout k.key -out c.crt` |
| Let's Encrypt | `sudo certbot --nginx` |

### Directive map

| Context | Directives seen in the tutorials |
| --- | --- |
| main | `worker_processes`, `user`, `include` |
| `events` | `worker_connections` |
| `http` | `include mime.types`, `upstream`, `proxy_cache_path`, `limit_req_zone`, `gzip*`, `server_tokens` |
| `server` | `listen`, `server_name`, `root`, `index`, `ssl_certificate`, `ssl_certificate_key`, `add_header`, `return` |
| `location` | `root`, `alias`, `try_files`, `return`, `rewrite`, `proxy_pass`, `proxy_set_header`, `proxy_cache`, `proxy_cache_valid`, `limit_req` |
| `upstream` | `server` (+ `weight`, `backup`), `least_conn`, `ip_hash` |

### Commands added in sections 10 to 19

| Task | Command |
| --- | --- |
| Dump the effective config | `sudo nginx -T` |
| Build flags and modules | `nginx -V 2>&1 \| tr ' ' '\n' \| grep -E 'module\|with-'` |
| Reopen logs / graceful stop / fast stop | `kill -USR1`, `kill -QUIT`, `kill -TERM` on the master PID |
| Which user, which permissions | `ps -o user= -C nginx`, `namei -l /path/to/file` |
| Test a vhost without DNS | `curl --resolve host:443:IP https://host/` |
| Show cache status | `curl -sI URL \| grep -i x-cache-status` |
| Certificate dates | `openssl s_client -connect host:443 -servername host </dev/null \| openssl x509 -noout -dates` |
| Renewal dry run (Docker) | `docker compose run --rm --entrypoint certbot certbot renew --dry-run` |
| Basic-auth user | `sudo htpasswd -c /etc/nginx/.htpasswd admin` |
| Metrics | `curl http://127.0.0.1:8081/nginx_status` |
| Validate on the compose network | `docker compose run --rm --no-deps --entrypoint nginx nginx -t` |
| Load test | `wrk -t4 -c100 -d30s --latency URL` |

### Assembled instructional baseline config (Docker, from sections 3 to 9)

```nginx
server_tokens off;
gzip on; gzip_min_length 1000; gzip_comp_level 5;
gzip_types text/css application/json application/javascript text/xml application/xml image/svg+xml;

proxy_cache_path /data/nginx/cache keys_zone=api_cache:10m max_size=1g inactive=30m use_temp_path=off;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

upstream backend_servers { server api:3001; server api2:3001; server api3:3001; }

# catch-all: unknown Host or direct IP access
server { listen 80 default_server; server_name _; return 444; }
server { listen 443 ssl default_server; server_name _; ssl_reject_handshake on; }

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    location / { proxy_pass http://web:3000; }

    location = /api/process-data {
        proxy_pass http://backend_servers/process-data;
        proxy_cache api_cache;
        proxy_cache_valid 200 10m;
        limit_req zone=api_limit;
    }
    location /api/ {
        proxy_pass http://backend_servers/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        limit_req zone=api_limit;
    }
    location /mongo/ {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;     # section 14; mount the file read-only
        proxy_set_header Authorization "";
        proxy_pass http://express:8081;
    }
}
```

This file integrates the tutorials; none of them ran it as a whole. It is an instructional baseline, not a finished production setup: add the renewal workflow (section 14), real-IP handling (section 12), monitoring (section 17) and failure handling (section 15), protect /mongo/ as shown, and test it with the smoke script of section 18. Run `nginx -t` before deploying. The `ssl_reject_handshake` line needs nginx 1.19.4 or newer.

### Exam-style questions

1. Explain the difference between a forward proxy and a reverse proxy, and name two benefits of putting nginx in front of a Node app.
2. A request to `/api/users` reaches the backend as `/api/users` and returns 404, but the app serves `/users`. Which one-character change fixes it, and why?
3. Compare `root` and `alias` in a `location` block. For `location /carbs` with `alias .../fruits`, which file is served for `/carbs/`?
4. What does `try_files /vegetables/veggies.html /index.html =404;` do, and why did the unconfigured `/vegetables` return 403 instead of 404?
5. Redirect versus rewrite: which changes the address bar, and what status code did the tutorial use for the redirect?
6. Why must `localhost` be replaced by service names in the nginx config once nginx runs in its own container? Which directory does the official image load config from?
7. Describe SSL termination. Give one argument for and one against terminating TLS at the proxy.
8. Write an `upstream` where `api` has weight 3, `api2` has weight 1 and `api3` is a backup, and the matching `proxy_pass`.
9. Caching `/api/` caused a created user not to appear. Explain the cause and the fix. What does `inactive=30m` do?
10. What is the effect of raising `gzip_comp_level` from 1 to 9, and why did the tutorial choose 5?
11. Why did adding `Content-Security-Policy: default-src 'self'` break the demo page?
12. What does `limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;` track, and roughly how many client states fit in the 10 MB zone? (Check the nginx docs.)
13. Why does `listen 80 default_server; server_name _; return 444;` not protect `https://SERVER_IP`, and how do you fix it?

### Exam-style questions, part B (sections 10 to 19)

1. Why does `worker_processes x worker_connections` overestimate how many clients a reverse proxy serves, and what else limits capacity?
2. In the request lifecycle, does `limit_req` run before or after `auth_basic`? What does that mean for unauthenticated floods?
3. Locations: `location = /`, `location /`, `location /docs/`, `location ^~ /static/`, `location ~* \.(png|jpg)$`. Which block serves `/static/logo.png`, `/docs/a.png` and `/index.html`? Explain the algorithm.
4. What is the difference between `rewrite ... last` and `rewrite ... break`? When does a rewrite redirect the browser?
5. After `docker compose up -d api` recreated the backend, nginx returns 502. Why? Give two fixes.
6. Explain `proxy_cache_bypass` versus `proxy_no_cache`. Which do you need for personalised responses, and why both?
7. Why does a WebSocket fail through a default `proxy_pass`? Which three directives fix it, and which timeout closes idle sockets?
8. Why is `set_real_ip_from 0.0.0.0/0;` dangerous?
9. A POST times out after the backend received it. With the default `proxy_next_upstream`, is it retried on another server? Why is that the right default?
10. Choose HTTP-01 or DNS-01 for (a) a wildcard certificate, (b) a server whose port 80 is closed. Why can `certbot --nginx` certificates not be renewed after nginx moves into Docker?
11. A location adds one `add_header` and the security headers set at server level disappear. Why, and how do you fix it?
12. Describe a safe HSTS rollout and why it is hard to undo.
13. What does status 499 mean, and what usually causes it?
14. Name three layers of health checking and the question each answers (nginx, application, dependency).

### Answer key

Part A:

1. A forward proxy sits on the client side (hides and controls clients); a reverse proxy sits on the server side. In front of Node it gives one public entry point with hidden backends, load balancing, TLS termination, caching and compression.
2. Add the trailing slash: `proxy_pass http://localhost:3001/;`. With a URI part, nginx replaces the matched location prefix (`/api/users` becomes `/users`); without it the full URI is forwarded.
3. `root` appends the full request URI to its path; `alias` replaces the matched prefix. For `location /carbs { alias .../fruits; }`, `/carbs/` is served from `.../fruits/index.html`.
4. It tries the listed paths in order and falls back to the last item (`=404`). `/vegetables` returned 403 because the directory exists but has no `index.html` and autoindex is off.
5. A redirect changes the address bar (the tutorial used `return 307 /fruits`); a rewrite changes the URI internally and the browser URL stays.
6. `localhost` inside the container is the container itself; use service names (`web`, `api`, `express`). The official image loads `/etc/nginx/conf.d/*.conf`.
7. TLS ends at nginx and plain HTTP runs to the backends. For: one certificate, central configuration, offloaded crypto. Against: unencrypted hop inside the network (fix with upstream TLS, section 14); backends cannot see client certificates unless nginx forwards them.
8. `upstream backend_servers { server api:3001 weight=3; server api2:3001 weight=1; server api3:3001 backup; }` with `proxy_pass http://backend_servers/;`.
9. The shared cache returned the stale `/api/users` list. Cache only the exact `/api/process-data` location. `inactive=30m` removes entries not accessed for 30 minutes, whether still valid or not.
10. A higher level gives smaller output with more CPU and latency and diminishing returns; level 5 is the compromise (default is 1).
11. `default-src 'self'` blocks the Bootstrap CSS from an external CDN; every external origin must be allowed explicitly.
12. It tracks clients by binary IP, in 10 MB of shared memory, at 5 requests per second. One megabyte holds about 16,000 states of 64 bytes (8,000 of 128 bytes on some platforms), so about 160,000 addresses; confirm in the nginx docs.
13. `listen 80 default_server` covers only HTTP. Add `listen 443 ssl default_server; ssl_reject_handshake on;` (nginx 1.19.4 or newer), or a certificate plus `return 444`.

Part B:

1. Each proxied request takes a client slot and an upstream slot, and each slot is a file descriptor, so capacity is roughly half and is capped by `worker_rlimit_nofile` and the OS limit.
2. Before. `limit_req` is in the pre-access phase, authentication in the access phase, so unauthenticated requests are counted and can be limited; but rejected requests still cost parsing work.
3. `/static/logo.png`: the `^~` block (it stops the regex search). `/docs/a.png`: the regex block (a regex beats a plain prefix). `/index.html`: `location /`. Algorithm: exact match, longest prefix (stop on `^~`), first matching regex in file order, else the remembered prefix.
4. `last` restarts location matching with the new URI; `break` stops rewriting and continues in the current location. Only `redirect`/`permanent` flags (or a replacement starting with a scheme) send the client to a new URL.
5. Nginx resolved `api` once at load and holds the old IP. Fixes: reload nginx, or use `resolver 127.0.0.11` with a variable in `proxy_pass` (or `server api:3001 resolve;` with a `zone` on nginx 1.27.3+).
6. Bypass skips reading from the cache; no-cache skips writing. Personalised responses need both with the same condition: otherwise a private response may be stored and served to others, or a bypassed private response is stored for later requests.
7. `Upgrade` and `Connection` are hop-by-hop and the backend connection is HTTP/1.0. Use `proxy_http_version 1.1;`, `proxy_set_header Upgrade $http_upgrade;`, `proxy_set_header Connection $connection_upgrade;`. `proxy_read_timeout` (default 60s) closes idle sockets.
8. Every client could claim any IP through `X-Forwarded-For`, defeating rate limits, allow/deny rules and logs.
9. No: non-idempotent methods are not retried after being sent, because the backend may already have executed them (double charge, double insert). `non_idempotent` overrides this and should be used only for idempotent endpoints.
10. (a) DNS-01 (wildcards need it). (b) DNS-01. The renewal config of `certbot --nginx` names the nginx authenticator, which cannot control a containerised nginx, and `--standalone` fails because the container holds port 80; re-issue with `--webroot` and a shared challenge volume.
11. `add_header` is array-type: defining any at an inner level discards all outer ones. Put the headers in an included snippet and include it in each location that sets its own.
12. Start with `max-age=300`, extend to a week, then months with `includeSubDomains` after every subdomain works over HTTPS, preload last. Browsers cache the policy for `max-age`, and broken HTTPS then gives a hard error users cannot click through.
13. Nginx-specific: the client closed the connection before the response was sent; typically a slow backend, an impatient client or a flaky mobile network.
14. Nginx liveness (`/healthz`: is the proxy answering), application readiness (`/readyz` proxied to the app), dependency health (the app's health endpoint checking database and queue).

### Sources

- Sarvin style coding, "Nginx complete course" (Docker, Compose, VPS, Certbot, caching, gzip, security): transcript supplied.
- Laith Harb, "NGINX crash course" (static content, MIME types, location, rewrites, load balancing): transcript supplied.
- TechWorld with Nana, "Nginx crash course" (concepts, proxy, load balancer, self-signed HTTPS): transcript supplied.
- Standards: RFC 9110, RFC 9111, RFC 9112, RFC 8446, RFC 8555, RFC 7239, RFC 1952, RFC 7932. Not opened in this session; cited by number from memory.

Open items to verify yourself: the exact `openssl` flags and the Dockerfiles are reconstructed from narration; Let's Encrypt renewal in Docker is not covered by the sources.

Sections 10 to 19 were written from the documented behaviour of nginx and from three review checklists you supplied; the commands and configurations were not executed here. Verify them against the nginx.org documentation for your exact version (`nginx -V`), and run the labs in section 19 on a test machine before production. The Ingress NGINX retirement was checked against the Kubernetes announcements of November 2025 and January 2026.
