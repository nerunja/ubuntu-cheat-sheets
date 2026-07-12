# Installing ttyd (Lightweight Web Terminal) on Ubuntu Server

ttyd is a single static binary (no runtime, no JVM, no database) that shares a terminal over a browser via websockets + xterm.js. This is the leanest option for "I just want a web shell into this VPS."

## 1. Download the prebuilt binary

No compiling needed — grab the static release binary directly from GitHub:

```bash
sudo curl -sSLo /usr/local/bin/ttyd \
  https://github.com/tsl0922/ttyd/releases/latest/download/ttyd.x86_64
sudo chmod +x /usr/local/bin/ttyd
ttyd --version
```

(If your VPS is ARM, use `ttyd.aarch64` instead of `ttyd.x86_64` — check `uname -m` first.)

## 2. Quick manual test (don't leave it like this)

```bash
ttyd -p 7681 login
```

Then browse to `http://<vps-ip>:7681` — you should get a login prompt in the browser using your system accounts. Ctrl+C to stop this test; we'll run it properly as a service next.

**Important:** never run `ttyd bash` or `ttyd sh` directly exposed to the internet — that gives an unauthenticated shell to anyone who hits the port. Always wrap it with `login` (so it asks for system credentials) and/or `-c user:pass` (HTTP basic auth), and bind it to localhost only, reverse-proxied through nginx with TLS. Details below.

## 3. Create a dedicated systemd service

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

Notes on flags:
- `-i 127.0.0.1` — only listens on localhost; not directly reachable from outside. nginx does the public-facing part.
- `-W` — allow the client to actually type/write to the TTY (ttyd is read-only by default).
- `login` — the command ttyd runs; this forces a proper username/password prompt against your system's PAM auth, same as a normal login.
- Running the service as `User=root` is necessary here so `login` can switch to whichever user authenticates — this is normal for this setup, since `login` itself does the privilege drop after auth.

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ttyd
sudo systemctl status ttyd
```

## 4. Put nginx in front with TLS (do this before opening any firewall port)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

`/etc/nginx/sites-available/ttyd`:

```nginx
server {
    listen 80;
    server_name terminal.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:7681;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600;   # keep long idle sessions alive
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/ttyd /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d terminal.yourdomain.com
```

Since you're on Dynu, use your Dynu DDNS hostname (or a subdomain CNAME'd to it) as `server_name` and for the certbot `-d` flag.

## 5. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# do NOT open 7681 — it's localhost-only and proxied through nginx
sudo ufw enable
```

## 6. (Optional) Extra layer: HTTP Basic Auth in front of the PAM login

If you want two layers of auth (nginx basic auth *and* the PAM login prompt), add to the nginx `location /` block:

```nginx
auth_basic "Restricted";
auth_basic_user_file /etc/nginx/.htpasswd;
```

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd yourusername
sudo systemctl reload nginx
```

## 7. Resource footprint check

```bash
systemctl status ttyd    # look at Memory: line
free -h
```

You should see ttyd sitting at a few MB — a night-and-day difference versus the Guacamole stack.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| Browser shows "connection refused" | ttyd only binds to 127.0.0.1 — check nginx is actually running and proxying, not trying to hit port 7681 directly from outside |
| WebSocket fails / terminal doesn't respond after connecting | Missing `Upgrade`/`Connection` headers in the nginx config — double check the block above |
| Login prompt appears but typing does nothing | Forgot `-W` flag — ttyd defaults to read-only |
| Works over HTTP but breaks after adding certbot | Certbot may have created a duplicate `listen 443` block — check `nginx -t` output and `/etc/nginx/sites-enabled/ttyd` for conflicts |
