# Geek Techniques Cheat Sheet

Lesser-known but very handy Linux / SSH / Git / Docker / nginx tricks, collected one at a time.
New techniques are appended at the end.

---

## Index

| # | Area | Technique |
|---|---|---|
| 1 | files | [rsync between two local folders](#1-rsync-between-two-local-folders) |
| 2 | git | [Git remote pointing at a folder or network path](#2-git-remote-pointing-at-a-folder-or-network-path) |
| 3 | nginx | [HTTPS and SSH on the same port 443 (stream + ssl_preread)](#3-https-and-ssh-on-the-same-port-443-stream--ssl_preread) |
| 4 | ssh | [SSH connection multiplexing](#4-ssh-connection-multiplexing) |
| 5 | shell | [Process substitution](#5-process-substitution) |
| 6 | git | [git worktree](#6-git-worktree) |
| 7 | docker | [nsenter into a container's network namespace](#7-nsenter-into-a-containers-network-namespace) |
| 8 | debugging | [Deleted-but-open files eating disk (lsof +L1)](#8-deleted-but-open-files-eating-disk-lsof-l1) |
| 9 | networking | [curl --resolve / --connect-to](#9-curl---resolve----connect-to) |
| 10 | nginx | [mirror: shadow live traffic to a new backend](#10-nginx-mirror-shadow-live-traffic-to-a-new-backend) |
| 11 | ssh | [SSH escape sequences](#11-ssh-escape-sequences) |
| 12 | git | [git bisect run](#12-git-bisect-run) |
| 13 | linux | [systemd-run: cap any command's memory/CPU](#13-systemd-run-cap-any-commands-memorycpu) |
| 14 | shell | [Edit the command in $EDITOR (Ctrl-X Ctrl-E, fc)](#14-edit-the-command-in-editor-ctrl-x-ctrl-e-fc) |
| 15 | networking | [socat: instant port forwarder / bridge](#15-socat-instant-port-forwarder--bridge) |
| 16 | docker | [docker diff](#16-docker-diff) |
| 17 | debugging | [strace a running process](#17-strace-a-running-process) |
| 18 | nginx | [nginx -T: dump the full merged config](#18-nginx--t-dump-the-full-merged-config) |
| 19 | ssh | [ProxyJump through a bastion](#19-proxyjump-through-a-bastion) |
| 20 | git | [git reflog: recover lost commits](#20-git-reflog-recover-lost-commits) |
| 21 | linux | [/proc/PID introspection + prlimit](#21-procpid-introspection--prlimit) |
| 22 | docker | [BuildKit cache mounts](#22-buildkit-cache-mounts) |
| 23 | shell | [xargs -P / -0 for safe parallel jobs](#23-xargs--p---0-for-safe-parallel-jobs) |
| 24 | debugging | [Where is a hung process stuck? (gdb, /proc stack, py-spy)](#24-where-is-a-hung-process-stuck) |
| 25 | networking | [ss filters and per-connection TCP stats](#25-ss-filters-and-per-connection-tcp-stats) |
| 26 | nginx | [Serve stale content when the backend is down](#26-serve-stale-content-when-the-backend-is-down) |
| 27 | git | [Pickaxe search and line history](#27-pickaxe-search-and-line-history) |
| 28 | ssh | [Reverse tunnel to a box behind NAT](#28-reverse-tunnel-to-a-box-behind-nat) |
| 29 | linux | [inotifywait: react to file changes](#29-inotifywait-react-to-file-changes) |
| 30 | docker | [The PID 1 signal problem](#30-the-pid-1-signal-problem) |
| 31 | shell | [Brace expansion + history shortcuts](#31-brace-expansion--history-shortcuts) |
| 32 | networking | [Debug TLS with openssl s_client](#32-debug-tls-with-openssl-s_client) |
| 33 | nginx | [Rate limiting with limit_req](#33-rate-limiting-with-limit_req) |
| 34 | debugging | [Find what crashed a server with journalctl](#34-find-what-crashed-a-server-with-journalctl) |
| 35 | git | [Fix an older commit with --fixup + --autosquash](#35-fix-an-older-commit-with---fixup----autosquash) |
| 36 | docker | [Compose healthcheck + depends_on: service_healthy](#36-compose-healthcheck--depends_on-service_healthy) |
| 37 | linux | [chattr +i / +a: immutable and append-only files](#37-chattr-i--a-immutable-and-append-only-files) |
| 38 | ssh | [ssh -D: dynamic SOCKS proxy](#38-ssh--d-dynamic-socks-proxy) |
| 39 | shell | [Bash strict mode + trap](#39-bash-strict-mode--trap) |
| 40 | networking | [Live remote packet capture into local Wireshark](#40-live-remote-packet-capture-into-local-wireshark) |
| 41 | nginx | [nginx auth_request: one login check in front of any app](#41-nginx-auth_request-one-login-check-in-front-of-any-app) |
| 42 | debugging | [eBPF one-shot tools (bcc / bpftrace)](#42-ebpf-one-shot-tools-bcc--bpftrace) |

---

## 1. rsync between two local folders

rsync is not just for remote copies. Locally it gives you resumable, incremental, verifiable copies.

```bash
rsync -aHAX --info=progress2 /src/ /dst/      # trailing slash on src = copy contents
rsync -a --delete /src/ /dst/                 # exact mirror (removes extras in dst)
rsync -an --delete /src/ /dst/                # dry run first (-n)
```

---

## 2. Git remote pointing at a folder or network path

A remote does not need a server. Any path to a bare repo works.

```bash
git init --bare /mnt/nas/repos/project.git
git remote add nas /mnt/nas/repos/project.git   # or file:///mnt/nas/repos/project.git
git push nas main
git clone /mnt/nas/repos/project.git
```

---

## 3. HTTPS and SSH on the same port 443 (stream + ssl_preread)

nginx `stream` peeks at the TLS ClientHello; SSH has none, so it falls to the default.

```nginx
stream {
    map $ssl_preread_protocol $upstream {
        ""      127.0.0.1:22;     # not TLS -> SSH
        default 127.0.0.1:8443;   # TLS -> https server
    }
    server {
        listen 443;
        proxy_pass $upstream;
        ssl_preread on;
    }
}
```

Move your `http { server { listen 443 ssl; } }` to `listen 8443 ssl;`.

---

## 4. SSH connection multiplexing

Reuse one TCP+auth connection for every later ssh/scp/rsync/git to the same host. Near-instant logins.

```sshconfig
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

```bash
ssh -O check host    # is a master running?
ssh -O exit host     # close it
```

---

## 5. Process substitution

`<(cmd)` turns a command's output into a file path.

```bash
diff <(sort a.txt) <(sort b.txt)
diff <(ssh web1 cat /etc/nginx/nginx.conf) <(ssh web2 cat /etc/nginx/nginx.conf)
comm -13 <(sort old.lst) <(sort new.lst)      # lines only in new
```

---

## 6. git worktree

Check out several branches side by side from one repo, no stash, no second clone.

```bash
git worktree add ../proj-hotfix hotfix/1.2
git worktree add -b feature/x ../proj-x main
git worktree list
git worktree remove ../proj-hotfix
```

---

## 7. nsenter into a container's network namespace

Use host tools (tcpdump, ss, curl) inside a container that has none.

```bash
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
sudo nsenter -t "$PID" -n ss -tlnp
sudo nsenter -t "$PID" -n tcpdump -i eth0 port 80
```

---

## 8. Deleted-but-open files eating disk (lsof +L1)

`df` says full, `du` says not. A process still holds a deleted file.

```bash
sudo lsof +L1                         # open files with link count 0
sudo lsof +L1 | grep -i log
: > /proc/<PID>/fd/<FD>               # truncate it without restarting the process
```

---

## 9. curl --resolve / --connect-to

Test a site on a specific server before (or without) changing DNS, with correct SNI and Host.

```bash
curl -v --resolve example.com:443:203.0.113.10 https://example.com/
curl -v --connect-to example.com:443:new-lb.internal:443 https://example.com/
```

---

## 10. nginx mirror: shadow live traffic to a new backend

Copy every request to a second backend; its responses are ignored.

```nginx
location / {
    mirror /mirror;
    mirror_request_body on;
    proxy_pass http://old_backend;
}
location = /mirror {
    internal;
    proxy_pass http://new_backend$request_uri;
}
```

---

## 11. SSH escape sequences

Type at the start of a line (press Enter first).

```text
~.    kill a hung session
~C    open ssh> prompt: -L 8080:localhost:80 adds a forward live
~#    list forwarded connections
~?    help
~~    send a literal ~ (nested ssh)
```

---

## 12. git bisect run

Let git binary-search the breaking commit automatically. Script exit 0 = good, 1-127 = bad, 125 = skip.

```bash
git bisect start HEAD v1.4.0
git bisect run ./test.sh            # or: git bisect run make test
git bisect reset
```

---

## 13. systemd-run: cap any command's memory/CPU

Run an ad-hoc command in its own cgroup with limits.

```bash
sudo systemd-run --scope -p MemoryMax=1G -p CPUQuota=50% ./heavy-job.sh
systemd-run --user --scope -p MemoryMax=500M make -j8
```

---

## 14. Edit the command in $EDITOR (Ctrl-X Ctrl-E, fc)

```text
Ctrl-X Ctrl-E   open the current command line in $EDITOR, runs on save
fc              open the previous command in $EDITOR
fc -l -10       list the last 10 commands
```

---

## 15. socat: instant port forwarder / bridge

```bash
socat TCP-LISTEN:8080,fork,reuseaddr TCP:10.0.0.5:80          # port forward
socat TCP-LISTEN:2375,fork,reuseaddr UNIX-CONNECT:/var/run/docker.sock   # TCP -> unix socket
socat - UNIX-CONNECT:/run/some.sock                            # talk to a socket by hand
```

---

## 16. docker diff

See which files a container changed compared to its image. `A` added, `C` changed, `D` deleted.

```bash
docker diff mycontainer
```

---

## 17. strace a running process

See what a live process is opening or connecting to.

```bash
sudo strace -f -e trace=openat,connect -p <PID>
sudo strace -f -e trace=network -s 200 -p <PID>
sudo strace -c -p <PID>                       # syscall summary, Ctrl-C to print
```

---

## 18. nginx -T: dump the full merged config

Tests the config and prints every file with all `include`s expanded.

```bash
sudo nginx -T
sudo nginx -T | grep -n server_name
```

---

## 19. ProxyJump through a bastion

```bash
ssh -J user@bastion user@10.0.0.12
scp -J bastion file.txt 10.0.0.12:/tmp/
```

```sshconfig
Host private-*
    ProxyJump bastion
```

Works transparently for scp, rsync and git once in `~/.ssh/config`.

---

## 20. git reflog: recover lost commits

Every HEAD move is logged locally, so "lost" commits after a bad reset/rebase/branch delete are still there.

```bash
git reflog
git reset --hard HEAD@{2}
git branch rescued <sha>
```

---

## 21. /proc/PID introspection + prlimit

```bash
tr '\0' '\n' < /proc/<PID>/environ     # its environment
ls -l /proc/<PID>/cwd /proc/<PID>/exe  # working dir and binary
cat /proc/<PID>/limits                 # ulimits it actually got
grep -E 'VmRSS|Threads' /proc/<PID>/status
sudo prlimit --pid <PID> --nofile=65535:65535   # raise limits on a running process
```

---

## 22. BuildKit cache mounts

Keep package caches across Docker builds without bloating the image.

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update && apt-get install -y build-essential
```

---

## 23. xargs -P / -0 for safe parallel jobs

```bash
find . -name '*.png' -print0 | xargs -0 -P "$(nproc)" -n 1 optipng
cat hosts.txt | xargs -P 10 -I{} ssh {} uptime
```

`-print0`/`-0` survives spaces and newlines in filenames.

---

## 24. Where is a hung process stuck?

```bash
sudo gdb -p <PID> -batch -ex 'thread apply all bt'   # native stack of every thread
sudo cat /proc/<PID>/stack                           # kernel-side stack
cat /proc/<PID>/wchan; echo                          # kernel function it's waiting in
sudo py-spy dump --pid <PID>                         # Python stack
```

---

## 25. ss filters and per-connection TCP stats

```bash
ss -tn state established '( dport = :443 or sport = :443 )'
ss -tn state time-wait | wc -l
ss -tlnp                                  # listening sockets with process
ss -ti dst 10.0.0.5                       # rtt, retrans, cwnd per connection
```

---

## 26. Serve stale content when the backend is down

```nginx
proxy_cache_path /var/cache/nginx keys_zone=app:10m max_size=1g inactive=1d;

location / {
    proxy_cache app;
    proxy_cache_valid 200 1s;                       # microcaching
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_background_update on;
    proxy_cache_lock on;                            # one request refreshes, rest wait
    proxy_pass http://backend;
}
```

---

## 27. Pickaxe search and line history

```bash
git log -S 'MAX_RETRIES' --oneline        # commits where the count of this string changed
git log -G 'timeout\s*=' -p               # commits whose diff matches a regex
git log -L 40,60:src/app.py               # history of a line range
git log -L :parse_config:src/config.py    # history of a function
```

---

## 28. Reverse tunnel to a box behind NAT

From the NATed box, expose its SSH on a public server.

```bash
ssh -N -R 2222:localhost:22 user@public-server
# then on public-server:
ssh -p 2222 user@localhost
```

Keep it up with autossh:

```bash
autossh -M 0 -N -o ServerAliveInterval=30 -o ServerAliveCountMax=3 \
  -R 2222:localhost:22 user@public-server
```

(Wrap it in a systemd service with `Restart=always` for reboots.)

---

## 29. inotifywait: react to file changes

```bash
inotifywait -m -e close_write,moved_to --format '%w%f' /srv/conf |
  while read -r f; do echo "changed: $f"; systemctl reload nginx; done

# raise limits if you watch many files
sudo sysctl fs.inotify.max_user_watches=524288
```

---

## 30. The PID 1 signal problem

PID 1 in a container ignores signals it has no handler for, so `docker stop` waits 10s then kills.

```bash
docker run --init myimage            # tiny init (tini) as PID 1, forwards signals, reaps zombies
```

```dockerfile
CMD ["node", "server.js"]            # exec form, not: CMD node server.js
```

```bash
# in entrypoint.sh, end with exec so your app becomes PID 1
exec "$@"
```

---

## 31. Brace expansion + history shortcuts

```bash
cp nginx.conf{,.bak}            # cp nginx.conf nginx.conf.bak
mkdir -p app/{src,tests,docs}
mv file.{txt,md}

!!            # last command (sudo !!)
!$            # last argument of the previous command
^old^new      # rerun last command replacing first "old"
!!:gs/a/b/    # rerun last command replacing all "a" with "b"
```

---

## 32. Debug TLS with openssl s_client

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer
openssl s_client -connect example.com:443 -tls1_2      # does it still accept TLS 1.2?
```

---

## 33. Rate limiting with limit_req

```nginx
http {
    geo $limited { default 1; 10.0.0.0/8 0; 192.168.1.10 0; }
    map $limited $limit_key { 0 ""; 1 $binary_remote_addr; }   # empty key = not limited
    limit_req_zone $limit_key zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_req_status 429;
            proxy_pass http://backend;
        }
    }
}
```

---

## 34. Find what crashed a server with journalctl

```bash
journalctl --list-boots                       # every boot the journal knows about
journalctl -b -1 -p err -n 50                 # last 50 errors from the previous boot
journalctl -b -1 -e                           # jump to the end of the previous boot (what happened just before)
journalctl -k -b -1 | grep -i -E 'oom|panic|i/o error'   # kernel: OOM kills, panics, disk errors
journalctl -b -1 --grep 'killed process'      # built-in regex filter
journalctl -u nginx --since '1 hour ago' -o cat   # message text only, no prefixes
journalctl _COMM=java -b -1                   # by process name (or _PID=1234)
```

If `--list-boots` only shows the current boot, the journal is in RAM only. Make it persistent:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

---

## 35. Fix an older commit with --fixup + --autosquash

Spotted a bug in a commit three commits back? Commit the fix marked as a fixup, and let rebase fold it in automatically. No manual reordering in the interactive editor.

```bash
git add -p
git commit --fixup=<sha>                  # creates "fixup! <original subject>"
git rebase -i --autosquash <sha>~1        # fixup is already moved and marked; just save
```

Extras:

```bash
git commit --fixup=:/'parse config'       # find the target commit by message instead of sha
git config --global rebase.autoSquash true   # plain "git rebase -i" then autosquashes
git commit --fixup=amend:<sha>            # also replace the target's commit message
git commit --fixup=reword:<sha>           # change only the message, no code
```

Only for commits you have not pushed (or on your own branch): it rewrites history.

---

## 36. Compose healthcheck + depends_on: service_healthy

`depends_on` alone only waits for the container to *start*, not to be *ready*. Add a healthcheck and wait on it.

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
  app:
    build: .
    depends_on:
      db:
        condition: service_healthy     # also: service_started, service_completed_successfully
```

```bash
docker compose ps                                  # shows (healthy) / (unhealthy)
docker inspect --format '{{json .State.Health}}' <container> | jq
```

---

## 37. chattr +i / +a: immutable and append-only files

File attributes that even root can't bypass without removing them first.

```bash
sudo chattr +i /etc/resolv.conf        # immutable: no edit, delete, rename, or link
sudo chattr +a /var/log/audit.log      # append-only: >> works, > and rm fail
lsattr /etc/resolv.conf                # ----i---------e------- /etc/resolv.conf
sudo chattr -i /etc/resolv.conf        # remove it again
```

> Classic use: stop NetworkManager/DHCP from overwriting `/etc/resolv.conf`. If `rm` says "Operation not permitted" as root, check `lsattr`.

---

## 38. ssh -D: dynamic SOCKS proxy

Turn any SSH server into a SOCKS5 proxy into its network.

```bash
ssh -N -D 1080 user@jumphost
curl --socks5-hostname localhost:1080 http://intranet.local/   # DNS resolved remotely too
```

Browser: set SOCKS5 proxy `localhost:1080` (and "proxy DNS when using SOCKS v5").

```sshconfig
Host work-proxy
    HostName jumphost
    DynamicForward 1080
```

---

## 39. Bash strict mode + trap

```bash
#!/usr/bin/env bash
set -Eeuo pipefail          # exit on error, unset vars, failed pipes; -E passes ERR trap into functions

tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT                                           # always clean up
trap 'echo "error on line $LINENO: $BASH_COMMAND" >&2' ERR          # say where it died
```

> Allow an expected failure explicitly: `grep foo file || true`.

---

## 40. Live remote packet capture into local Wireshark

Capture on a server, watch live in Wireshark on your laptop, no pcap file copying.

```bash
ssh user@server 'sudo tcpdump -i eth0 -U -s0 -w - not port 22' | wireshark -k -i -
```

`-U` flushes each packet, `-w -` writes pcap to stdout, `not port 22` skips your own SSH traffic. Needs passwordless sudo for tcpdump (or capabilities on it).

---

## 41. nginx auth_request: one login check in front of any app

nginx sends a subrequest to an auth service; 2xx lets the request through, 401/403 blocks it.

```nginx
location / {
    auth_request /_auth;
    auth_request_set $user $upstream_http_x_user;
    proxy_set_header X-User $user;
    proxy_pass http://app;
}

location = /_auth {
    internal;
    proxy_pass http://auth-service/verify;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
}

error_page 401 = @login;
location @login { return 302 /login?rd=$request_uri; }
```

> This is how oauth2-proxy / Authelia sit in front of apps that have no login of their own.

---

## 42. eBPF one-shot tools (bcc / bpftrace)

See system-wide activity live with near-zero overhead.

```bash
sudo apt install bpfcc-tools bpftrace     # tools are named *-bpfcc on Ubuntu

sudo execsnoop-bpfcc            # every new process, with args (catches short-lived ones)
sudo opensnoop-bpfcc -p <PID>   # files being opened
sudo tcpconnect-bpfcc           # outbound TCP connections, by process
sudo biolatency-bpfcc           # disk I/O latency histogram

sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
```
