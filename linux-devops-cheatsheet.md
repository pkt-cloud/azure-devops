# Linux Commands for DevOps — Daily Use, Troubleshooting & Interviews

A practical Linux command reference for **day-to-day DevOps work**, **production troubleshooting**, **disk/CPU/memory investigation**, **networking**, **services**, **cron jobs**, and **common interview questions**.

> Recommended for DevOps Engineers, Cloud Engineers, SREs, System Administrators, and Linux interview preparation.

---

## Table of Contents

- [1. Navigation and File Operations](#1-navigation-and-file-operations)
- [2. Viewing and Editing Files](#2-viewing-and-editing-files)
- [3. Searching Files and Text](#3-searching-files-and-text)
- [4. Disk Space Troubleshooting](#4-disk-space-troubleshooting)
- [5. CPU Spike Troubleshooting](#5-cpu-spike-troubleshooting)
- [6. Memory Troubleshooting](#6-memory-troubleshooting)
- [7. Process Management](#7-process-management)
- [8. Service Management with systemd](#8-service-management-with-systemd)
- [9. Logs and journalctl](#9-logs-and-journalctl)
- [10. Networking](#10-networking)
- [11. DNS Troubleshooting](#11-dns-troubleshooting)
- [12. File Permissions](#12-file-permissions)
- [13. Users and Groups](#13-users-and-groups)
- [14. SSH and Remote Copy](#14-ssh-and-remote-copy)
- [15. Cron Jobs](#15-cron-jobs)
- [16. Package Management](#16-package-management)
- [17. Archive and Compression](#17-archive-and-compression)
- [18. Text Processing](#18-text-processing)
- [19. Environment Variables](#19-environment-variables)
- [20. OS and System Information](#20-os-and-system-information)
- [21. Disk and Mount Troubleshooting](#21-disk-and-mount-troubleshooting)
- [22. Application and Port Troubleshooting](#22-application-and-port-troubleshooting)
- [23. CPU Spike Interview Workflow](#23-cpu-spike-interview-workflow)
- [24. Memory Spike Interview Workflow](#24-memory-spike-interview-workflow)
- [25. Server Slow Troubleshooting](#25-server-slow-troubleshooting)
- [26. Disk IO Troubleshooting](#26-disk-io-troubleshooting)
- [27. Open Files and File Descriptors](#27-open-files-and-file-descriptors)
- [28. Common Linux Interview Topics](#28-common-linux-interview-topics)
- [29. Common Linux Interview Questions](#29-common-linux-interview-questions)
- [30. Daily DevOps Command Set](#30-daily-devops-command-set)

---

## 1. Navigation and File Operations

| Task | Command |
|---|---|
| Show current directory | `pwd` |
| List files | `ls` |
| Detailed file listing | `ls -lh` |
| Include hidden files | `ls -la` |
| Change directory | `cd /path` |
| Go to home directory | `cd ~` |
| Go one directory up | `cd ..` |
| Return to previous directory | `cd -` |
| Create directory | `mkdir test` |
| Create nested directories | `mkdir -p app/logs/archive` |
| Create an empty file | `touch file.txt` |
| Copy a file | `cp source.txt target.txt` |
| Copy a directory | `cp -r source/ destination/` |
| Move or rename | `mv old.txt new.txt` |
| Delete a file | `rm file.txt` |
| Delete a directory | `rm -rf folder/` |
| Check file type | `file filename` |
| Locate an executable | `which python` |
| Identify command type | `type ls` |
| Open command manual | `man grep` |

> **Warning:** Be very careful when using `rm -rf`, especially as `root`.

---

## 2. Viewing and Editing Files

| Task | Command |
|---|---|
| Display complete file | `cat file.txt` |
| First 10 lines | `head file.txt` |
| First 50 lines | `head -50 file.txt` |
| Last 10 lines | `tail file.txt` |
| Last 100 lines | `tail -100 file.txt` |
| Follow file/log live | `tail -f app.log` |
| Follow last 100 lines | `tail -100f app.log` |
| Page through file | `less file.txt` |
| Edit using Vim | `vi file.txt` |
| Edit using Nano | `nano file.txt` |

Common production command:

```bash
tail -f /var/log/application.log
```

---

## 3. Searching Files and Text

| Task | Command |
|---|---|
| Find file by name | `find / -name "file.txt"` |
| Ignore case | `find / -iname "file.txt"` |
| Find log files | `find /var/log -name "*.log"` |
| Files modified in last day | `find /var/log -mtime -1` |
| Files larger than 1 GB | `find / -type f -size +1G` |
| Search text | `grep "ERROR" app.log` |
| Ignore case | `grep -i "error" app.log` |
| Recursive search | `grep -R "database" /etc/` |
| Show line numbers | `grep -n "ERROR" app.log` |
| Exclude matching lines | `grep -v "DEBUG" app.log` |
| Multiple patterns | `grep -E "ERROR|WARN" app.log` |

Useful troubleshooting pattern:

```bash
grep -iE "error|failed|exception|critical" application.log
```

---

## 4. Disk Space Troubleshooting

Check filesystem usage:

```bash
df -h
```

Check inode usage:

```bash
df -i
```

A filesystem can have free disk space but still fail to create files if all inodes are consumed.

Find largest directories:

```bash
du -sh /*
```

Inside `/var`:

```bash
du -sh /var/*
```

Sort by largest usage:

```bash
du -sh /var/* | sort -hr
```

Show top 20 largest files/directories:

```bash
du -ah /var | sort -hr | head -20
```

Find files larger than 1 GB:

```bash
find / -type f -size +1G 2>/dev/null
```

Find files larger than 500 MB:

```bash
find / -type f -size +500M 2>/dev/null
```

Check log usage:

```bash
du -sh /var/log/*
```

Check systemd journal usage:

```bash
journalctl --disk-usage
```

Remove journal entries older than seven days:

```bash
sudo journalctl --vacuum-time=7d
```

Limit journal storage to approximately 500 MB:

```bash
sudo journalctl --vacuum-size=500M
```

Find deleted files that are still open by a process:

```bash
sudo lsof +L1
```

### Important interview scenario

A large log file was deleted, but disk space was not released because a process still had the deleted file open.

```bash
sudo lsof +L1
```

After identifying the process, investigate and restart it only if operationally safe.

---

## 5. CPU Spike Troubleshooting

Interactive CPU/process view:

```bash
top
```

If available:

```bash
htop
```

Top CPU-consuming processes:

```bash
ps aux --sort=-%cpu | head
```

Detailed top CPU processes:

```bash
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head
```

Number of CPUs:

```bash
nproc
```

CPU details:

```bash
lscpu
```

Load average:

```bash
uptime
```

Per-CPU utilization:

```bash
mpstat -P ALL 1
```

Process-level CPU statistics:

```bash
pidstat 1
```

Specific PID:

```bash
pidstat -p 1234 1
```

Threads consuming CPU:

```bash
top -H -p 1234
```

---

## 6. Memory Troubleshooting

Check memory:

```bash
free -h
```

Monitor continuously:

```bash
watch free -h
```

Processes using the most memory:

```bash
ps aux --sort=-%mem | head
```

Detailed version:

```bash
ps -eo pid,user,%mem,%cpu,cmd --sort=-%mem | head
```

Check swap:

```bash
swapon --show
```

Detailed memory information:

```bash
cat /proc/meminfo
```

Check Out-of-Memory events:

```bash
dmesg | grep -i oom
```

or:

```bash
journalctl -k | grep -i oom
```

Check whether the OOM killer terminated a process:

```bash
dmesg | grep -i "out of memory"
```

```bash
journalctl -k | grep -i "killed process"
```

> Linux uses available memory for caching. Low `free` memory alone does not necessarily mean the server has a memory problem. Pay attention to the `available` value from `free -h`.

---

## 7. Process Management

Show all processes:

```bash
ps aux
```

Find a process:

```bash
ps aux | grep nginx
```

Better option:

```bash
pgrep nginx
```

Inspect a PID:

```bash
ps -fp 1234
```

Process tree:

```bash
pstree -p
```

Graceful termination:

```bash
kill 1234
```

Force termination:

```bash
kill -9 1234
```

Kill process by name:

```bash
pkill nginx
```

Run command in background:

```bash
command &
```

List shell jobs:

```bash
jobs
```

Return job to foreground:

```bash
fg
```

Keep a process running after SSH disconnect:

```bash
nohup ./script.sh > app.log 2>&1 &
```

---

## 8. Service Management with systemd

Check service:

```bash
systemctl status nginx
```

Start:

```bash
sudo systemctl start nginx
```

Stop:

```bash
sudo systemctl stop nginx
```

Restart:

```bash
sudo systemctl restart nginx
```

Reload configuration:

```bash
sudo systemctl reload nginx
```

Enable on boot:

```bash
sudo systemctl enable nginx
```

Disable on boot:

```bash
sudo systemctl disable nginx
```

Check whether enabled:

```bash
systemctl is-enabled nginx
```

Show failed services:

```bash
systemctl --failed
```

Show running services:

```bash
systemctl --type=service --state=running
```

---

## 9. Logs and journalctl

Service logs:

```bash
journalctl -u nginx
```

Follow live:

```bash
journalctl -u nginx -f
```

Last 100 lines:

```bash
journalctl -u nginx -n 100
```

Logs since today:

```bash
journalctl --since today
```

Last hour:

```bash
journalctl --since "1 hour ago"
```

Specific period:

```bash
journalctl \
  --since "2026-09-09 09:00" \
  --until "2026-09-09 10:00"
```

Kernel logs:

```bash
journalctl -k
```

Traditional Linux log locations include:

```text
/var/log/syslog
/var/log/messages
/var/log/auth.log
/var/log/secure
```

Exact files depend on the Linux distribution.

---

## 10. Networking

Show IP addresses:

```bash
ip addr
```

Short form:

```bash
ip a
```

Routing table:

```bash
ip route
```

Network interfaces:

```bash
ip link
```

Connectivity test:

```bash
ping example.com
```

DNS lookup:

```bash
nslookup example.com
```

Detailed DNS lookup:

```bash
dig example.com
```

Query a specific DNS server:

```bash
dig @8.8.8.8 example.com
```

HTTP connectivity:

```bash
curl https://example.com
```

Headers only:

```bash
curl -I https://example.com
```

Verbose TLS/network troubleshooting:

```bash
curl -vk https://example.com
```

Check TCP port connectivity:

```bash
nc -vz server.example.com 443
```

Example for SQL Server:

```bash
nc -vz database.example.com 1433
```

Listening ports:

```bash
ss -tulpn
```

Listening TCP ports only:

```bash
ss -ltnp
```

Find process using port 8080:

```bash
sudo lsof -i :8080
```

or:

```bash
sudo ss -ltnp | grep 8080
```

Trace network route:

```bash
traceroute example.com
```

---

## 11. DNS Troubleshooting

Configured DNS servers:

```bash
cat /etc/resolv.conf
```

Resolve hostname:

```bash
nslookup server.example.com
```

or:

```bash
dig server.example.com
```

Check local host overrides:

```bash
cat /etc/hosts
```

Check hostname:

```bash
hostname
```

Fully qualified hostname:

```bash
hostname -f
```

---

## 12. File Permissions

Check permissions:

```bash
ls -l
```

Change permissions:

```bash
chmod 755 script.sh
```

Make executable:

```bash
chmod +x script.sh
```

Change owner:

```bash
chown user file.txt
```

Change owner and group:

```bash
chown user:group file.txt
```

Recursive ownership:

```bash
chown -R appuser:appgroup /opt/app
```

Permission values:

| Permission | Value |
|---|---:|
| Read | 4 |
| Write | 2 |
| Execute | 1 |
| `rwx` | 7 |
| `rw-` | 6 |
| `r-x` | 5 |

Example:

```bash
chmod 755 file
```

means:

```text
Owner: rwx
Group: r-x
Other: r-x
```

---

## 13. Users and Groups

Current user:

```bash
whoami
```

User ID and groups:

```bash
id
```

Logged-in users:

```bash
who
```

Login history:

```bash
last
```

Create user:

```bash
sudo useradd username
```

Set password:

```bash
sudo passwd username
```

Create group:

```bash
sudo groupadd devops
```

Add user to group:

```bash
sudo usermod -aG devops username
```

Switch user:

```bash
su - username
```

Run command with elevated privileges:

```bash
sudo command
```

---

## 14. SSH and Remote Copy

Connect to server:

```bash
ssh user@server
```

Use SSH private key:

```bash
ssh -i key.pem user@server
```

Use custom SSH port:

```bash
ssh -p 2222 user@server
```

Copy file to remote host:

```bash
scp file.txt user@server:/tmp/
```

Copy directory:

```bash
scp -r folder user@server:/tmp/
```

Synchronize files:

```bash
rsync -av source/ user@server:/destination/
```

SSH client config:

```text
~/.ssh/config
```

SSH server config:

```text
/etc/ssh/sshd_config
```

---

## 15. Cron Jobs

Cron is used to schedule commands and scripts.

Edit current user's crontab:

```bash
crontab -e
```

List jobs:

```bash
crontab -l
```

Remove all jobs:

```bash
crontab -r
```

Edit root cron:

```bash
sudo crontab -e
```

Cron format:

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

Every day at 2 AM:

```cron
0 2 * * * /opt/scripts/backup.sh
```

Every 5 minutes:

```cron
*/5 * * * * /opt/scripts/check.sh
```

Monday to Friday at 9 AM:

```cron
0 9 * * 1-5 /opt/scripts/report.sh
```

Every Sunday at midnight:

```cron
0 0 * * 0 /opt/scripts/cleanup.sh
```

Write stdout and stderr to a log:

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Check cron service on Ubuntu/Debian:

```bash
systemctl status cron
```

On RHEL/CentOS/Rocky/AlmaLinux:

```bash
systemctl status crond
```

Cron logs:

```bash
grep CRON /var/log/syslog
```

or:

```bash
journalctl -u cron
```

---

## 16. Package Management

### Ubuntu / Debian

```bash
sudo apt update
sudo apt upgrade
sudo apt install nginx
sudo apt remove nginx
apt list --installed
```

### RHEL / Rocky / AlmaLinux

```bash
sudo dnf update
sudo dnf install nginx
sudo dnf remove nginx
```

Older systems may use:

```bash
yum install nginx
```

---

## 17. Archive and Compression

Create tar archive:

```bash
tar -cvf archive.tar folder/
```

Create compressed tar.gz:

```bash
tar -czvf archive.tar.gz folder/
```

Extract tar.gz:

```bash
tar -xzvf archive.tar.gz
```

List archive contents:

```bash
tar -tzvf archive.tar.gz
```

Create zip:

```bash
zip -r archive.zip folder/
```

Extract zip:

```bash
unzip archive.zip
```

---

## 18. Text Processing

Sort lines:

```bash
sort file.txt
```

Remove duplicate lines:

```bash
sort file.txt | uniq
```

Count lines:

```bash
wc -l file.txt
```

Print first field:

```bash
awk '{print $1}' file.txt
```

Example process information:

```bash
ps aux | awk '{print $2,$3,$11}'
```

Replace text:

```bash
sed 's/old/new/g' file.txt
```

Print lines 10 through 20:

```bash
sed -n '10,20p' file.txt
```

Extract first field separated by `:`:

```bash
cut -d':' -f1 /etc/passwd
```

---

## 19. Environment Variables

Display environment:

```bash
env
```

Display PATH:

```bash
echo $PATH
```

Set temporary environment variable:

```bash
export APP_ENV=production
```

Check it:

```bash
echo $APP_ENV
```

User-level persistent settings are commonly placed in:

```text
~/.bashrc
```

Apply changes:

```bash
source ~/.bashrc
```

---

## 20. OS and System Information

Kernel information:

```bash
uname -a
```

Linux distribution:

```bash
cat /etc/os-release
```

Hostname information:

```bash
hostnamectl
```

System uptime:

```bash
uptime
```

Last boot:

```bash
who -b
```

CPU information:

```bash
lscpu
```

Memory:

```bash
free -h
```

Block devices:

```bash
lsblk
```

Mounted filesystems:

```bash
mount
```

Filesystem type and utilization:

```bash
df -Th
```

---

## 21. Disk and Mount Troubleshooting

Block devices:

```bash
lsblk
```

Filesystem UUIDs:

```bash
blkid
```

Persistent mount configuration:

```bash
cat /etc/fstab
```

Current mounts:

```bash
mount
```

Mount manually:

```bash
sudo mount /dev/sdb1 /data
```

Unmount:

```bash
sudo umount /data
```

Find process blocking unmount:

```bash
lsof /data
```

or:

```bash
fuser -vm /data
```

---

## 22. Application and Port Troubleshooting

Suppose an application expected on port `8080` is unavailable.

### Step 1 — Check process

```bash
ps aux | grep application
```

### Step 2 — Check service

```bash
systemctl status application
```

### Step 3 — Check listening port

```bash
ss -ltnp | grep 8080
```

### Step 4 — Test locally

```bash
curl -v http://localhost:8080
```

### Step 5 — Check logs

```bash
journalctl -u application -n 100
```

### Step 6 — Check CPU and memory

```bash
top
```

### Step 7 — Check disk

```bash
df -h
```

### Step 8 — Check remote connectivity

```bash
nc -vz server 8080
```

This is a strong troubleshooting sequence to explain in interviews.

---

## 23. CPU Spike Interview Workflow

If asked:

> Production server CPU is at 100%. How would you troubleshoot it?

Start with:

```bash
uptime
top
ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
pidstat 1
top -H -p PID
journalctl -u application
```

Possible causes include:

- Application process
- High request traffic
- Infinite loop
- Database processing
- Java garbage collection
- Compression or encryption
- Backup job
- Cron job
- Container workload
- Kernel activity
- Disk I/O wait

Use:

```bash
vmstat 1
```

Important columns:

| Column | Meaning |
|---|---|
| `r` | Runnable processes |
| `si` | Swap in |
| `so` | Swap out |
| `us` | User CPU |
| `sy` | System CPU |
| `id` | Idle CPU |
| `wa` | I/O wait |

High `wa` can indicate a disk/storage bottleneck rather than a pure CPU issue.

---

## 24. Memory Spike Interview Workflow

Use:

```bash
free -h
ps aux --sort=-%mem | head
top
vmstat 1
```

Inspect a process:

```bash
pmap -x PID | tail
```

Check OOM events:

```bash
dmesg | grep -i oom
```

Remember: Linux often uses free memory for filesystem cache. Look at **available memory**, swap activity, process usage, and OOM events before concluding there is a memory leak.

---

## 25. Server Slow Troubleshooting

Useful initial health checks:

```bash
uptime
top
free -h
df -h
df -i
vmstat 1
iostat -xz 1
ss -s
```

Check errors:

```bash
journalctl -p err
```

Recent kernel events:

```bash
dmesg | tail -100
```

---

## 26. Disk IO Troubleshooting

Basic disk statistics:

```bash
iostat
```

Detailed utilization:

```bash
iostat -xz 1
```

Process-level disk I/O:

```bash
iotop
```

Important metrics include:

- `%util`
- `await`
- `r/s`
- `w/s`

High `%util` together with high `await` can indicate storage contention.

---

## 27. Open Files and File Descriptors

List open files:

```bash
lsof
```

Files opened by a PID:

```bash
lsof -p PID
```

Find process using port 443:

```bash
lsof -i :443
```

Deleted files still held open:

```bash
lsof +L1
```

Current shell file descriptor limit:

```bash
ulimit -n
```

Process limits:

```bash
cat /proc/PID/limits
```

---

## 28. Common Linux Interview Topics

| Topic | Commands / Concepts |
|---|---|
| CPU | `top`, `ps`, `mpstat`, `pidstat` |
| Memory | `free`, `vmstat`, `/proc/meminfo` |
| Disk | `df`, `du`, `lsblk` |
| Disk I/O | `iostat`, `iotop` |
| Networking | `ip`, `ss`, `curl`, `nc` |
| DNS | `dig`, `nslookup` |
| Processes | `ps`, `kill`, `pgrep` |
| Logs | `journalctl`, `tail`, `grep` |
| Services | `systemctl` |
| Scheduling | `cron`, `crontab` |
| Permissions | `chmod`, `chown` |
| SSH | `ssh`, `scp`, `rsync` |
| Search | `find`, `grep` |
| Text processing | `awk`, `sed`, `cut` |
| Packages | `apt`, `dnf`, `yum` |
| Kernel logs | `dmesg` |
| Open files | `lsof` |

---

## 29. Common Linux Interview Questions

### What is the difference between `df` and `du`?

```text
df = filesystem-level disk usage
du = file/directory-level disk usage
```

### What is the difference between a hard link and a symbolic link?

Hard link:

```bash
ln file hardlink
```

Symbolic link:

```bash
ln -s file softlink
```

A hard link points to the same inode. A symbolic link points to a pathname.

### What is the difference between `kill PID` and `kill -9 PID`?

```bash
kill PID
```

Sends `SIGTERM`, allowing the process to shut down gracefully.

```bash
kill -9 PID
```

Sends `SIGKILL`, which cannot be handled or ignored by the process.

Use `SIGKILL` only when required.

### What is the difference between `>` and `>>`?

Overwrite:

```bash
command > file
```

Append:

```bash
command >> file
```

### What does `2>&1` mean?

```bash
command > app.log 2>&1
```

It redirects standard error (`stderr`) to the same destination as standard output (`stdout`).

### How do you find which process is listening on port 443?

```bash
ss -ltnp | grep :443
```

### How do you find the highest CPU-consuming process?

```bash
ps aux --sort=-%cpu | head
```

### How do you find the highest memory-consuming process?

```bash
ps aux --sort=-%mem | head
```

### How do you find the largest files/directories?

```bash
du -ah / | sort -hr | head
```

---

## 30. Daily DevOps Command Set

These are the commands worth practicing until they become automatic:

```bash
pwd
ls -lah
cd
cp
mv
rm
mkdir
cat
less
head
tail -f

grep
find
awk
sed

df -h
df -i
du -sh
du -sh * | sort -hr

free -h
top
ps aux
uptime
vmstat
iostat

systemctl status
systemctl restart
journalctl

ip addr
ip route
ss -ltnp
curl -vk
nc -vz
nslookup
dig

chmod
chown

ssh
scp
rsync

crontab -e
crontab -l

tar
zip
unzip

lsof
dmesg
lsblk
mount
```

---

## Recommended Priority for DevOps Engineers

Focus especially on these areas:

1. `systemctl` and service troubleshooting
2. `journalctl`, `tail`, and `grep` for log analysis
3. `curl`, `ss`, `nc`, `dig`, and `nslookup` for networking
4. `df`, `du`, `find`, and `lsof` for disk issues
5. `top`, `ps`, `free`, `vmstat`, `pidstat`, and `iostat` for performance issues
6. `chmod`, `chown`, users, groups, and SSH
7. `cron` and scheduled automation
8. Shell pipelines using `grep`, `awk`, `sed`, `sort`, and `cut`

---

## Quick Production Health Check

When you connect to an unhealthy Linux server and need a fast first look:

```bash
uptime
free -h
df -h
df -i
top
ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
ss -ltnp
systemctl --failed
journalctl -p err -n 50
```

This quickly tells you whether the issue is related to **CPU, memory, disk, ports, services, or system errors**.

---

## License

Feel free to use, modify, and expand this cheat sheet for learning, DevOps practice, and interview preparation.
