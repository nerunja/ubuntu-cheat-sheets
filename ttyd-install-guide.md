# Installing ttyd on Ubuntu Laptop (Home Server Setup)

ttyd is a single static binary that shares a terminal over a browser via websockets + xterm.js. This guide is specifically for setting up ttyd on a **laptop or home server** that's behind a home router with IPv6 support and Cloudflare DNS.

## Prerequisites

- Ubuntu laptop/server with IPv6 connectivity
- Nginx already installed and running (for reverse proxy + SSL)
- Cloudflare account for DNS management
- Domain name pointing to your home network via Cloudflare

## 1. Download the prebuilt binary

```bash
sudo curl -sSLo /usr/local/bin/ttyd \
  https://github.com/tsl0922/ttyd/releases/latest/download/ttyd.x86_64
sudo chmod +x /usr/local/bin/ttyd
ttyd --version
```

(If your laptop is ARM, use `ttyd.aarch64` instead.)

## 2. Create a dedicated systemd service

`/etc/systemd/system/ttyd.service`:

```ini
[Unit]
Description=ttyd - web terminal
After=network.target

[Service]
# Bind to localhost only — nginx will handle external access + TLS
ExecStart=/usr/local/bin/ttyd -i 127.0.0.1 -p 7681 -W login
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ttyd
sudo systemctl status ttyd
```

## 3. Configure nginx with SSL (Cloudflare + IPv6 setup)

### Important: IPv6 Considerations for Home Networks

Home IPv6 addresses are often **dynamic/temporary** (with `mngtmpaddr` flag). Don't use them directly in DNS AAAA records unless you have a stable address. Instead, let nginx listen on all IPv6 interfaces and use Cloudflare proxy or port forwarding.

### Create nginx config

`/etc/nginx/sites-available/xps-tty-ssl.conf` (replace `xps-tty.itekk.in` with your domain):

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name xps-tty.itekk.in;
    return 301 https://$server_name$request_uri;
}

# HTTPS with SSL
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name xps-tty.itekk.in;

    # SSL certificate (will be configured by certbot)
    ssl_certificate /etc/letsencrypt/live/xps-tty.itekk.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xps-tty.itekk.in/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:7681;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 3600;   # keep long idle sessions alive
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/xps-tty-ssl.conf /etc/nginx/sites-enabled/
```

## 4. Get SSL certificate with certbot

Since the SSL config references certificates that don't exist yet, use a temporary HTTP-only config for certbot:

### Option A: Temporary HTTP config (recommended)

```bash
# Create temporary HTTP-only config
sudo tee /etc/nginx/sites-available/xps-tty-http.conf > /dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name xps-tty.itekk.in;
    
    location / {
        proxy_pass http://127.0.0.1:7681;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600;
    }
}
EOF

# Temporarily use HTTP config
sudo rm -f /etc/nginx/sites-enabled/xps-tty-ssl.conf
sudo ln -s /etc/nginx/sites-available/xps-tty-http.conf /etc/nginx/sites-enabled/xps-tty.conf
sudo nginx -t && sudo systemctl reload nginx
```

### Run certbot

```bash
sudo certbot --nginx -d xps-tty.itekk.in
```

Certbot will automatically update your nginx config to use the new SSL certificate.

### Option B: If using Cloudflare (alternative)

If you have Cloudflare proxy enabled (orange cloud), you may need to:
1. Temporarily disable proxy (gray cloud) in Cloudflare DNS
2. Run certbot
3. Re-enable proxy after certificate is issued

Or use DNS challenge instead:
```bash
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /path/to/cloudflare.ini -d xps-tty.itekk.in
```

## 5. Configure Cloudflare DNS

### For IPv6 setup:
1. In Cloudflare DNS, add an **AAAA record** for your subdomain
2. Point it to your **home's IPv6 address** (check with `ip -6 addr show`)
3. **Warning:** Home IPv6 addresses are often dynamic. Consider:
   - Using IPv4 with port forwarding instead (more stable)
   - Setting up dynamic DNS update for IPv6
   - Using Cloudflare proxy (orange cloud) which handles IPv6->IPv4 translation

### For IPv4 setup (more reliable for home networks):
1. Add an **A record** pointing to your home's public IPv4
2. Set up port forwarding on your router (port 80/443 → laptop's local IP)
3. This is more stable than IPv6 for most home networks

## 6. Verify setup

```bash
# Check ttyd is running
sudo systemctl status ttyd

# Check nginx config
sudo nginx -t

# Test local connection
curl -I http://localhost:7681

# Test HTTPS (after certbot)
curl -I https://your-domain.com
```

## 7. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# do NOT open 7681 — it's localhost-only and proxied through nginx
sudo ufw enable
```

## Troubleshooting

| Symptom | Likely cause | Solution |
|---|---|---|
| Certbot fails with Cloudflare IPv6 in error | Cloudflare proxy intercepting challenges | Disable orange cloud temporarily, run certbot, re-enable |
| "Cannot load certificate" nginx error | SSL cert doesn't exist yet | Use temporary HTTP config for certbot (see Step 4) |
| IPv6 address changes / stops working | Home IPv6 is dynamic (mngtmpaddr) | Use IPv4 with port forwarding, or set up dynamic DNS |
| Browser shows router login page | Wrong IPv6 in AAAA record, or router intercepting | Verify AAAA record points to laptop, not router |
| Works locally but not remotely | Firewall or port forwarding issue | Check ufw, router port forwarding, Cloudflare settings |
| SSL cert domain mismatch | Certificate issued for wrong domain | Re-run certbot with correct `-d` flag |

## Notes

- **IPv6 on home networks:** Most home IPv6 addresses are temporary/dynamic. If you need stable remote access, consider IPv4 with port forwarding instead.
- **Cloudflare proxy:** The orange cloud proxy can simplify setup by handling IPv6->IPv4 translation and providing CDN benefits.
- **Multiple services:** If you're running multiple services (like router admin, jellyfin, etc.), follow the nginx proxy pattern shown in this guide.
