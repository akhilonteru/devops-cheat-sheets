# Nginx Cheat Sheet

> Nginx web server & reverse proxy reference — commands, core configuration, proxying, load balancing, SSL, and security hardening.

---

## Table of Contents

- [Commands](#1-commands)
- [Configuration Structure](#2-configuration-structure)
- [Static File Serving](#3-static-file-serving)
- [Reverse Proxy](#4-reverse-proxy)
- [Load Balancing](#5-load-balancing)
- [SSL / TLS](#6-ssl--tls)
- [Rewrites & Redirects](#7-rewrites--redirects)
- [Security Hardening](#8-security-hardening)
- [Rate Limiting](#9-rate-limiting)
- [Logging & Status](#10-logging--status)
- [Docker Usage](#11-docker-usage)

---

## 1. Commands

```
nginx -t                        # Test configuration syntax (always before reload!)
nginx -s reload                 # Graceful reload (zero downtime)
nginx -s stop                   # Fast shutdown
nginx -s quit                   # Graceful shutdown
systemctl start|stop|restart|enable nginx
nginx -T                        # Dump full effective configuration
nginx -c /etc/nginx/nginx.conf  # Start with specific config
```

## 2. Configuration Structure

```
/etc/nginx/
├── nginx.conf                  # Global config
├── conf.d/*.conf               # Included server blocks
├── sites-available/            # Debian/Ubuntu style
└── sites-enabled/              # Symlinks to sites-available
```

```nginx
# nginx.conf (global)
user  nginx;
worker_processes  auto;         # or fixed number (CPU cores)
pid   /var/run/nginx.pid;
error_log  /var/log/nginx/error.log warn;

events {
    worker_connections  1024;   # connections per worker process
    use epoll;                  # connection processing method (Linux)
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    tcp_nopush      on;
    keepalive_timeout  65;
    client_max_body_size 50m;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1024;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

## 3. Static File Serving

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Prevent dotfiles access
    location ~ /\. {
        deny all;
    }
}
```

## 4. Reverse Proxy

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;               # app server
        proxy_http_version 1.1;

        # Essential headers
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSockets
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Timeouts & buffering
    proxy_connect_timeout 5s;
    proxy_read_timeout    60s;
    proxy_send_timeout    60s;

    # Health endpoint proxied to backend
    location /healthz {
        proxy_pass http://127.0.0.1:3000/healthz;
        access_log off;
    }
}
```

**Path handling:** `proxy_pass http://backend;` (no URI) passes the original path; `proxy_pass http://backend/app/;` (with URI) rewrites the prefix.

## 5. Load Balancing

```nginx
upstream backend {
    # least_conn;              # Fewest active connections
    # ip_hash;                 # Session persistence by client IP
    # hash $request_uri consistent;   # Consistent hashing
    server 10.0.0.11:8080 weight=3 max_fails=2 fail_timeout=30s;
    server 10.0.0.12:8080;
    server 10.0.0.13:8080 backup;      # Only if others down
    keepalive 32;                      # Idle keepalive connections to upstream
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

**Methods:** `round_robin` (default), `least_conn`, `ip_hash`, `hash key [consistent]`, `random`.

## 6. SSL / TLS

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;              # Force HTTPS
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    # HSTS (only when HTTPS is stable)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    root /var/www/example;
    index index.html;
}
```

```bash
# Let's Encrypt with certbot
certbot --nginx -d example.com -d www.example.com
certbot renew --dry-run
```

## 7. Rewrites & Redirects

```nginx
# Redirect old URL
location /old-page {
    return 301 /new-page;
}

# Redirect whole domain
server {
    listen 80;
    server_name old.com;
    return 301 $scheme://new.com$request_uri;
}

# Friendly URLs
location /blog {
    rewrite ^/blog/(\d+)$ /posts?id=$1 last;      # 'last' re-searches locations
}

# Trailing slash normalization
rewrite ^([^.]*[^/])$ $1/ permanent;
```

**Codes:** `301` permanent, `302` temporary, `return 403/404` directly.

## 8. Security Hardening

```nginx
server {
    # Hide version
    server_tokens off;

    # Security headers
    add_header X-Frame-Options           "SAMEORIGIN" always;
    add_header X-Content-Type-Options    "nosniff" always;
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy   "default-src 'self'" always;

    # Basic auth
    location /admin {
        auth_basic           "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    # Allow only specific IPs
    location /internal {
        allow 10.0.0.0/8;
        deny all;
        proxy_pass http://backend;
    }

    # Limit request body
    client_max_body_size 10m;
}
```

```bash
htpasswd -c /etc/nginx/.htpasswd admin        # Create user (install apache2-utils)
```

## 9. Rate Limiting

```nginx
# In http block:
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    location /login {
        limit_req zone=perip burst=20 nodelay;     # Allow bursts of 20
        proxy_pass http://backend;
    }

    location /download {
        limit_rate 512k;                            # Per-connection bandwidth cap
        limit_conn addr 5;                          # Max 5 concurrent conns per IP
    }
}
```

## 10. Logging & Status

```nginx
server {
    access_log /var/log/nginx/example.access.log main;
    error_log  /var/log/nginx/example.error.log warn;

    # Disable logging for noise
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    # stub_status (bind to localhost, protect it)
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```
curl -s http://127.0.0.1/nginx_status
# Active connections: 291
# server accepts handled requests: 16630948 16630948 31070465
# Reading: 6 Writing: 145 Waiting: 140
```

```bash
# Top IPs by request count
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Status code distribution
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

## 11. Docker Usage

```bash
docker run -d --name nginx \
  -p 80:80 -p 443:443 \
  -v ./nginx.conf:/etc/nginx/nginx.conf:ro \
  -v ./sites:/etc/nginx/conf.d:ro \
  -v ./html:/usr/share/nginx/html:ro \
  -v /etc/letsencrypt:/etc/letsencrypt:ro \
  nginx:alpine

docker exec nginx nginx -t        # Test config inside container
docker exec nginx nginx -s reload
```

```dockerfile
# Minimal static image
FROM nginx:alpine
COPY html/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---
