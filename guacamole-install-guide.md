# Installing Apache Guacamole 1.6.0 on Ubuntu Server (VPS)

## 0. Before you start — resource reality check

Guacamole is made of three moving parts, all running at once:

| Component | Role | Typical idle RAM |
|---|---|---|
| `guacd` | Native daemon, talks RDP/VNC/SSH to remote hosts | ~10–15 MB |
| Tomcat 9 | Java servlet container serving the web client | ~150–250 MB |
| MySQL/MariaDB (optional but recommended) | Stores users/connections | ~150–300 MB |

If your VPS has 512 MB–1 GB RAM total, this stack alone can eat most of it — especially Tomcat (JVM) and MySQL. Options if RAM is tight:

- Skip the database, use the simple `user-mapping.xml` file-based auth (no MySQL needed) — saves ~200 MB.
- Add a swapfile if you don't have one (`fallocate -l 1G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`, then add to `/etc/fstab`).
- Consider whether you actually need full RDP/VNC gatewaying, or whether you just want a **web SSH terminal into this one VPS** — if it's only the latter, something like `ttyd` or Cockpit's terminal is far lighter than Guacamole. Worth a gut-check before building the whole stack.

Assuming you still want Guacamole, here's the build.

---

## 1. Update system and install guacd build dependencies

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential libcairo2-dev libjpeg-turbo8-dev \
  libpng-dev libtool-bin uuid-dev libavcodec-dev libavformat-dev \
  libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev \
  libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev \
  libpulse-dev libssl-dev libvorbis-dev libwebp-dev libwww-perl \
  ghostscript
```

If any package name doesn't resolve (varies slightly between 22.04/24.04), drop it and continue — most are optional codec/protocol support, not hard requirements.

## 2. Download and build guacamole-server (provides `guacd`)

```bash
cd /tmp
GUAC_VERSION="1.6.0"
wget "https://downloads.apache.org/guacamole/${GUAC_VERSION}/source/guacamole-server-${GUAC_VERSION}.tar.gz"
tar -xzf guacamole-server-${GUAC_VERSION}.tar.gz
cd guacamole-server-${GUAC_VERSION}/
```

On Ubuntu 24.04, OpenSSL 3.x throws deprecation warnings that `configure` treats as fatal errors — disable that:

```bash
CFLAGS=-Wno-error ./configure --with-init-dir=/etc/init.d --with-systemd-dir=/etc/systemd/system
```

Check the "Library status" / "Protocol support" summary it prints — confirm SSH, RDP, VNC show `yes` for whichever protocols you need. Then build:

```bash
make -j$(nproc)
sudo make install
sudo ldconfig
```

Enable and start guacd:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now guacd
sudo systemctl status guacd
```

You should see `guacd` listening on `127.0.0.1:4822`.

## 3. Install Tomcat 9 and deploy the Guacamole web client

```bash
sudo apt install -y tomcat9
```

Download the prebuilt client WAR (no need to compile Java):

```bash
GUAC_VERSION="1.6.0"
wget "https://downloads.apache.org/guacamole/${GUAC_VERSION}/binary/guacamole-${GUAC_VERSION}.war" -O /tmp/guacamole.war
sudo cp /tmp/guacamole.war /var/lib/tomcat9/webapps/guacamole.war
```

## 4. Create the Guacamole config directory

```bash
sudo mkdir -p /etc/guacamole/{extensions,lib}
sudo ln -s /etc/guacamole /usr/share/tomcat9/.guacamole
```

Create `/etc/guacamole/guacamole.properties`:

```bash
sudo tee /etc/guacamole/guacamole.properties > /dev/null <<'EOF'
guacd-hostname: localhost
guacd-port: 4822

# File-based auth (simple, no DB needed)
user-mapping: /etc/guacamole/user-mapping.xml
EOF
```

## 5. Simple auth: user-mapping.xml (skip if using MySQL below)

```bash
sudo tee /etc/guacamole/user-mapping.xml > /dev/null <<'EOF'
<user-mapping>
    <authorize username="admin" password="CHANGE_THIS_PASSWORD">
        <connection name="My VPS SSH">
            <protocol>ssh</protocol>
            <param name="hostname">127.0.0.1</param>
            <param name="port">22</param>
            <param name="username">yourlinuxuser</param>
        </connection>
    </authorize>
</user-mapping>
EOF
sudo chmod 640 /etc/guacamole/user-mapping.xml
sudo chown root:tomcat /etc/guacamole/user-mapping.xml
```

This is fine for a single-user setup on your own VPS. Change `CHANGE_THIS_PASSWORD` immediately.

### (Optional, production-grade) MySQL auth instead

Only do this if you want multi-user management, connection history, and the admin UI's user/connection editor — and only if your RAM budget allows it.

```bash
sudo apt install -y mariadb-server mariadb-client
sudo mysql_secure_installation

# Get the Guacamole JDBC auth extension + schema
GUAC_VERSION="1.6.0"
wget "https://downloads.apache.org/guacamole/${GUAC_VERSION}/binary/guacamole-auth-jdbc-${GUAC_VERSION}.tar.gz" -O /tmp/jdbc.tar.gz
tar -xzf /tmp/jdbc.tar.gz -C /tmp
sudo cp /tmp/guacamole-auth-jdbc-${GUAC_VERSION}/mysql/guacamole-auth-jdbc-mysql-${GUAC_VERSION}.jar /etc/guacamole/extensions/

# MySQL Connector/J
sudo apt install -y libmariadb-java
sudo ln -s /usr/share/java/mariadb-java-client.jar /etc/guacamole/lib/

sudo mysql -u root -p <<'EOF'
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'STRONG_PASSWORD_HERE';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'localhost';
FLUSH PRIVILEGES;
EOF

cat /tmp/guacamole-auth-jdbc-${GUAC_VERSION}/mysql/schema/*.sql | sudo mysql -u root -p guacamole_db
```

Add to `/etc/guacamole/guacamole.properties`:

```
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: STRONG_PASSWORD_HERE
```

If you go this route, remove the `user-mapping:` line — the two auth methods can coexist but it's cleaner to pick one.

## 6. Restart Tomcat and guacd

```bash
sudo systemctl restart tomcat9 guacd
sudo systemctl status tomcat9
```

Give Tomcat 20–30 seconds to deploy the WAR on first start.

## 7. Test it (before touching nginx/firewall)

```bash
curl -I http://localhost:8080/guacamole/
```

Should return `HTTP/1.1 200`. From your own machine, open:

```
http://<your-vps-ip>:8080/guacamole/
```

Login: `admin` / whatever you set (file-based), or `guacadmin` / `guacadmin` (MySQL default — **change immediately**).

## 8. Put nginx in front with SSL (recommended — don't expose raw port 8080)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

`/etc/nginx/sites-available/guacamole`:

```nginx
server {
    listen 80;
    server_name guacamole.yourdomain.com;

    location / {
        proxy_pass http://localhost:8080/guacamole/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d guacamole.yourdomain.com
```

Since you're on Dynu, you likely already have a dynamic-DNS hostname (Dynu is a DDNS provider) — point that hostname at `guacamole.yourdomain.com` or just use your Dynu-provided domain directly as `server_name`.

## 9. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 8080/tcp    # block direct Tomcat access from outside
sudo ufw enable
```

## 10. Post-install hardening

- Change/delete the default `guacadmin` account if you used MySQL auth.
- Restrict `/etc/guacamole/user-mapping.xml` permissions (already done above — `640`, owned by `root:tomcat`).
- Consider fail2ban on the nginx access log for repeated failed Guacamole logins.
- If RAM is genuinely tight, watch `free -h` and `systemctl status tomcat9` after a day of uptime — Tomcat's JVM heap can grow; you can cap it in `/etc/default/tomcat9` via `JAVA_OPTS="-Xmx256m"`.

---

### Quick troubleshooting

| Symptom | Likely cause |
|---|---|
| `guacd` fails to start | Check `journalctl -u guacd -n 50`; usually a missing lib from step 1 |
| Tomcat 200s but blank page | WAR not fully deployed yet, or GUACAMOLE_HOME symlink missing (step 4) |
| "Cannot connect to guacd" in browser | `guacd` not running, or `guacd-hostname`/`port` mismatch in guacamole.properties |
| SSH connection fails with auth error | Guacamole's SSH client only supports certain key/cipher types — check `libssh2` was built with all algorithms; also confirm `ssh-rsa` isn't disabled server-side if using key auth |
