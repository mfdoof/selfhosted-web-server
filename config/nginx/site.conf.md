# Nginx — Hardened Site Config

**Place at:** `/etc/nginx/sites-available/<your-domain>` (then symlink into `sites-enabled/`)

Reverse-proxy config for the Network Health Manager. Serves the React static build and proxies the `/api/` namespace to Uvicorn. Replace `nethealth.example.com` with your domain.

**Key points:**

- The Cloudflare Tunnel delivers HTTP (not HTTPS) to port 80 internally; Cloudflare handles HTTP→HTTPS at the edge. **Do not** add a port 80 → 443 redirect here, or you'll create an infinite redirect loop.
- Real client IP is resolved from the tunnel's `X-Forwarded-For` header — without it, every request looks like `127.0.0.1` and breaks rate limiting and fail2ban.
- Port 443 carries the Cloudflare IP whitelist as defense in depth.

```nginx
limit_req_zone $binary_remote_addr zone=general:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=expensive:10m rate=10r/m;

# Block direct IP access — drops the connection with no response.
server {
    listen 80 default_server;
    server_name _;
    return 444;
}

server {
    listen 80;
    server_name nethealth.example.com;

    # Resolve real client IP from the Cloudflare tunnel (forwards from 127.0.0.1).
    set_real_ip_from 127.0.0.1;
    real_ip_header X-Forwarded-For;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    root /var/www/nethealth.example.com;
    index index.html;

    location / {
        limit_req zone=general burst=10 nodelay;
        try_files $uri $uri/ /index.html;
    }

    # Expensive API routes — tighter rate limit
    location ~ ^/api/(hosts/[^/]+/(scan|ping|analyze)|dns/(lookup|analyze))$ {
        limit_req zone=expensive burst=5 nodelay;
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # All other API routes
    location /api/ {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    server_name nethealth.example.com;
    server_tokens off;

    ssl_certificate /etc/letsencrypt/live/nethealth.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nethealth.example.com/privkey.pem;

    # Allow the local tunnel
    allow 127.0.0.1;

    # Cloudflare IP whitelist — blocks non-Cloudflare traffic on port 443.
    # Source: https://www.cloudflare.com/ips/ (refresh periodically).
    allow 173.245.48.0/20;
    allow 103.21.244.0/22;
    allow 103.22.200.0/22;
    allow 103.31.4.0/22;
    allow 141.101.64.0/18;
    allow 108.162.192.0/18;
    allow 190.93.240.0/20;
    allow 188.114.96.0/20;
    allow 197.234.240.0/22;
    allow 198.41.128.0/17;
    allow 162.158.0.0/15;
    allow 104.16.0.0/13;
    allow 104.24.0.0/14;
    allow 172.64.0.0/13;
    allow 131.0.72.0/22;
    deny all;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    set_real_ip_from 127.0.0.1;
    real_ip_header X-Forwarded-For;

    root /var/www/nethealth.example.com;
    index index.html;

    location / {
        limit_req zone=general burst=10 nodelay;
        try_files $uri $uri/ /index.html;
    }

    location ~ ^/api/(hosts/[^/]+/(scan|ping|analyze)|dns/(lookup|analyze))$ {
        limit_req zone=expensive burst=5 nodelay;
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
