---
tags: [devops, devops/linux]
aliases: [Linux, Unix Commands, Shell, Bash, Linux Commands, Command Line, systemd, grep sed awk]
updated: 2026-09-27
---
# Linux & Shell Essentials

> [!tldr] TL;DR
> Almost every server, container and CI runner runs Linux, so the shell is a core engineering skill. Learn to **navigate and manipulate files**, understand **permissions and users**, inspect and control **processes**, compose small tools with **pipes and redirection** (`grep`, `sed`, `awk`, `sort`, `xargs`), work remotely with **ssh**, manage services with **systemd/journald**, and debug **networking** (`ss`, `curl`, `dig`). Everything is a file, and small tools combine into powerful pipelines.

## 1. Files and navigation

The original quick-reference table from this note:

| Command | Example | Description |
| --- | --- | --- |
| `cd` | `$ cd ..` | cd one directory up |
| `cd` | `$ cd ~` or just `$ cd` | cd to home directory |
| `cd` | `$ cd ~/environment/` | cd to path including home dir |
| `cd <dirname>` | `$ cd environment/` | change directory |
| `cat <file>` | `$ cat ~/.ssh/id_rsa.pub` | concatenate and display file contents |
| `cp <source> <target>` | `$ cp foo bar` | copy file |
| `ls` | `$ ls -l` | list contents |
| `mkdir <dirname>` | `$ mkdir environment` | make directory |
| `mv <source> <target>` | `$ mv foo bar` | move file (rename) |
| `rmdir <directory>` | `$ rmdir environment/` | remove empty directory |
| `rm <file>` | `$ rm foo` | remove file |
| `rm -rf <directory>` | `$ rm -rf tmp/` | remove nonempty directory (no undo!) |

More you will use daily:

```bash
pwd                          # where am I
ls -lah                      # long list, all (incl. dotfiles), human sizes
cd -                         # back to previous directory
mkdir -p a/b/c               # create nested dirs
cp -r src/ backup/           # copy directory recursively
touch notes.txt              # create empty file / update mtime
ln -s /opt/app/current app   # symbolic link
less big.log                 # page through (q to quit, / to search)
head -n 20 file; tail -n 50 file
tail -f /var/log/app.log     # follow a growing file
find . -name "*.log" -mtime +7 -size +10M      # find by name/age/size
find . -name "*.tmp" -delete
du -sh * | sort -h           # what is using disk in this dir
df -h                        # free space per filesystem
tar -czf backup.tgz dir/ && tar -xzf backup.tgz
which node; type ll          # where a command comes from
```

Filesystem layout (FHS): `/etc` config, `/var/log` logs, `/var/lib` app state, `/home` users, `/tmp` temporary, `/usr/bin` programs, `/opt` add-on software, `/proc` and `/sys` kernel views (everything is a file).

## 2. Permissions and users

```text
-rwxr-x---  1 deploy www  4096 Sep 27 10:00 deploy.sh
│└┬┘└┬┘└┬┘      owner  group
│ │  │  └ others: ---
│ │  └ group: r-x
│ └ owner: rwx
└ type: - file, d directory, l link
```

| Symbol | Octal | On a file | On a directory |
|---|---|---|---|
| r | 4 | read content | list names |
| w | 2 | modify | create/delete entries |
| x | 1 | execute | enter (`cd`), access entries |

```bash
chmod 755 deploy.sh          # rwx r-x r-x
chmod u+x script.sh          # add execute for owner
chmod 600 ~/.ssh/id_ed25519  # private key: owner read/write only (ssh insists)
chown deploy:www app/ -R     # change owner and group recursively
id; groups                   # who am I, which groups
sudo -u postgres psql        # run as another user
umask                        # default permission mask for new files
```

Special bits: **setuid/setgid** (run as file owner/group), **sticky bit** on `/tmp` (only owners can delete their files). Principle of least privilege: services run as dedicated non-root users; use `sudo` for specific commands, not a root shell. Extra layers: ACLs (`getfacl`/`setfacl`), SELinux/AppArmor, Linux capabilities.

## 3. Processes

```bash
ps aux | grep node           # snapshot of processes
pgrep -a node                # find by name
top / htop                   # live view (CPU, memory, load)
kill <pid>                   # SIGTERM (15): ask nicely
kill -9 <pid>                # SIGKILL: last resort, no cleanup
pkill -f "node server.js"
nohup ./job.sh > job.log 2>&1 &   # background, survives logout
jobs; fg %1; bg %1           # shell job control (Ctrl+Z suspends)
nice -n 10 ./batch.sh        # lower priority
lsof -i :8080                # which process holds port 8080
lsof -p <pid>                # files a process has open
uptime                       # load average (1, 5, 15 min)
free -h                      # memory (look at "available")
vmstat 1; iostat -xz 1       # CPU/memory/IO over time
strace -p <pid>              # system calls (why is it stuck?)
```

Key concepts: every process has a PID, parent (PPID), user, and exit code (0 = success). **Signals** (`SIGTERM`, `SIGKILL`, `SIGHUP` often = reload config, `SIGINT` = Ctrl+C). **Zombies** are finished children not yet reaped by the parent; **orphans** get re-parented to PID 1. Containers are processes too ([[Docker Basics]]).

## 4. Pipes, redirection and text processing

```bash
cmd > out.txt          # stdout to file (overwrite)
cmd >> out.txt         # append
cmd 2> err.txt         # stderr to file
cmd > all.txt 2>&1     # both
cmd < input.txt        # stdin from file
cmd1 | cmd2            # stdout of cmd1 → stdin of cmd2
cmd | tee out.txt      # show and save
$(cmd)                 # command substitution
cmd1 && cmd2           # run cmd2 only if cmd1 succeeded;  || only if it failed
```

### grep, sed, awk and friends

```bash
grep -rn "TODO" src/                     # recursive, with line numbers
grep -i "error" app.log | grep -v "health"   # case-insensitive, exclude
grep -E "5[0-9]{2}" access.log           # extended regex
grep -c "timeout" app.log                # count
rg "useEffect" --type ts                 # ripgrep: faster, respects .gitignore

sed -n '100,120p' big.log                # print lines 100-120
sed -i 's/localhost/db.internal/g' app.conf   # in-place replace (GNU sed; macOS: sed -i '')
sed '/^#/d' config.ini                   # delete comment lines

awk '{print $1}' access.log              # first column
awk -F, '$3 > 100 {print $1, $3}' data.csv   # CSV filter
awk '{sum += $NF} END {print sum}' sizes.txt # sum last column

cut -d: -f1 /etc/passwd                  # field 1, ":" delimiter
sort | uniq -c | sort -rn | head         # frequency count (classic!)
wc -l file                               # count lines
xargs -n1 -P4 curl -sO < urls.txt        # run a command per input line, 4 in parallel
jq '.items[] | {name: .metadata.name}' pods.json   # JSON processing
```

Worked example: top 10 IPs hitting 5xx errors in an nginx log:

```bash
awk '$9 ~ /^5/ {print $1}' /var/log/nginx/access.log \
  | sort | uniq -c | sort -rn | head -10
```

### Shell scripting basics

```bash
#!/usr/bin/env bash
set -euo pipefail             # exit on error, unset vars, and failures inside pipes
IFS=$'\n\t'

readonly ENV="${1:?usage: deploy.sh <env>}"
log() { printf '%s %s\n' "$(date -Is)" "$*" >&2; }
TMPDIR_JOB="$(mktemp -d)"
trap 'log "cleaning up"; rm -rf "$TMPDIR_JOB"' EXIT   # runs on any exit, even errors

for svc in api worker; do
  log "deploying $svc to $ENV"
  if ! ./deploy-one.sh "$svc" "$ENV"; then
    log "failed: $svc"; exit 1
  fi
done
```

Always quote variables (`"$var"`), use `shellcheck` in CI, and move to Python/Go when scripts grow beyond ~100 lines.

## 5. SSH

```bash
ssh-keygen -t ed25519 -C "you@example.com"     # modern key type
ssh-copy-id deploy@server                       # install public key
ssh deploy@server
ssh -i ~/.ssh/key -p 2222 user@host
scp file.txt user@host:/tmp/; rsync -avz --delete dist/ user@host:/var/www/
ssh -L 5432:db.internal:5432 bastion            # local port forward: localhost:5432 → private DB
ssh -J bastion user@private-host                # jump host
```

```text
# ~/.ssh/config
Host prod-bastion
  HostName bastion.example.com
  User deploy
  IdentityFile ~/.ssh/id_ed25519
Host prod-*
  ProxyJump prod-bastion
```

Server hardening: disable password and root login (`PasswordAuthentication no`, `PermitRootLogin no`), keys or SSO-issued short-lived certificates, fail2ban, or avoid SSH entirely with SSM Session Manager / IAP / Teleport.

## 6. systemd and logs

```bash
systemctl status nginx
sudo systemctl start|stop|restart|reload nginx
sudo systemctl enable --now myapp      # start now and at boot
systemctl list-units --type=service --failed
journalctl -u myapp -f                 # follow a service's logs
journalctl -u myapp --since "1 hour ago" -p err
journalctl -k                          # kernel messages (OOM killer shows here)
```

A minimal unit file:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My API
After=network-online.target
Wants=network-online.target

[Service]
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node server.js
Environment=NODE_ENV=production
Restart=on-failure
RestartSec=3
# sandboxing
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now myapp
```

Timers (`.timer` units) are the systemd alternative to cron ([[Cron Jobs]]).

## 7. Networking tools

```bash
ip a; ip r                         # interfaces/addresses; routes
ss -tulpn                          # listening TCP/UDP sockets + process (replaces netstat)
curl -v https://api.example.com/health        # full request/response + TLS handshake
curl -s -o /dev/null -w "%{http_code} %{time_total}\n" https://example.com
dig +short example.com; dig example.com MX    # DNS
nslookup example.com
ping -c 4 host; traceroute host (or mtr host) # reachability / path
nc -zv db.internal 5432                        # is the port open?
openssl s_client -connect example.com:443 -servername example.com   # inspect TLS cert
sudo tcpdump -i any port 443 -w cap.pcap       # capture packets (open in Wireshark)
cat /etc/hosts; cat /etc/resolv.conf           # local name resolution config
```

Debug order for "service X cannot reach Y": DNS resolves? → route/security group/firewall allows? → port listening on Y (`ss`)? → TLS valid? → HTTP status/app logs.

> [!info]+ 🆕 2025–2026
> - **Ubuntu 25.10 (October 2025)** made **sudo-rs** (a memory-safe Rust reimplementation of `sudo`) the default sudo provider and switched core utilities to **Rust coreutils (uutils)**; it also shipped Linux 6.17 and systemd 257. Most flags behave the same, but edge-case differences exist in scripts; test automation on new images.
> - **Kubernetes v1.35 (2025-12)** made the kubelet refuse to start on **cgroup v1** by default, so older distros without cgroup v2 are effectively unsupported for Kubernetes nodes (see [[Kubernetes Workloads in Production]]).

## Pitfalls

> [!warning] Common pitfalls
> - `rm -rf $DIR/` with an unset `DIR` → deletes `/`. Use `set -u` and `${DIR:?}`.
> - `chmod -R 777` to "fix" permissions: a security hole. Find the right owner/group instead.
> - `kill -9` as the first reaction: no graceful shutdown, possible corruption.
> - Parsing `ls` output in scripts; use `find -print0 | xargs -0` or globs.
> - Forgetting that `cmd | grep x` returns the pipeline's last exit status (use `set -o pipefail`).
> - Editing config on a server by hand instead of through [[Infrastructure as Code]] or images.

## Senior perspective

- Treat servers as cattle: debug with these tools, but fix through code, images and automation.
- Know the performance "60-second checklist": `uptime`, `dmesg | tail`, `vmstat 1`, `mpstat -P ALL 1`, `pidstat 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `sar -n TCP,ETCP 1`, `top` (Brendan Gregg).
- eBPF tools (`bpftrace`, BCC, Cilium/Pixie) give deep production visibility with low overhead.
- In containers, many tools are missing: use ephemeral debug containers (`kubectl debug`) or `nsenter` from the node.

## Interview questions

> [!question]- What do permissions 755 and 644 mean?
> 755: owner rwx, group r-x, others r-x (typical for executables/directories). 644: owner rw-, group r--, others r-- (typical for files). Each digit is read(4)+write(2)+execute(1).

> [!question]- Difference between SIGTERM and SIGKILL?
> SIGTERM (15) asks a process to terminate and can be caught for graceful cleanup. SIGKILL (9) cannot be caught or ignored; the kernel stops the process immediately.

> [!question]- How do you find which process is listening on port 8080?
> `ss -tulpn | grep 8080` or `lsof -i :8080`.

> [!question]- A server is slow. What do you check?
> Load average vs CPU count (`uptime`), CPU saturation and steal (`top`, `mpstat`), memory and swapping (`free`, `vmstat`), disk I/O wait (`iostat`), network errors (`ss`, `sar -n`), kernel messages (`dmesg` for OOM kills), then the application's own metrics and logs.

> [!question]- What does `set -euo pipefail` do?
> `-e` exits on any command failure, `-u` treats unset variables as errors, `-o pipefail` makes a pipeline fail if any command in it fails (not just the last). Together they make bash scripts fail fast instead of continuing in a broken state.

> [!question]- Count the most common error messages in a log file.
> `grep ERROR app.log | awk -F'ERROR' '{print $2}' | sort | uniq -c | sort -rn | head`

> [!question]- What is systemd?
> The init system (PID 1) on most Linux distributions: it starts and supervises services defined in unit files, handles dependencies, restarts, logging (journald), timers and resource control via cgroups.

## Related
- [[DevOps Index]] · [[Docker Basics]] · [[Containers vs VMs vs Serverless]] · [[Git Essentials & Workflows]] · [[Cron Jobs]] · [[Nginx]]
- [[Rails Project Structure]] (the other half of the original note)
- Original note: [[Unix Commands And Rails Structure (original)]]

## Sources
- [GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/) · [GNU Bash manual](https://www.gnu.org/software/bash/manual/)
- [The Linux man-pages project](https://man7.org/linux/man-pages/)
- [systemd documentation](https://systemd.io/) · [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [OpenSSH manual pages](https://www.openssh.com/manual.html)
- [Brendan Gregg: Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [ShellCheck](https://www.shellcheck.net/)
- [Ubuntu 25.10 release notes](https://documentation.ubuntu.com/release-notes/25.10/)
- William Shotts, *The Linux Command Line* (free online)
