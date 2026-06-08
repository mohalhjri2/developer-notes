# Linux — The Complete Field Guide

> "On a UNIX system, everything is a file. If something is not a file, it is a process."

---

## Table of Contents

1. [File System](#file-system)
2. [Navigation & Files](#navigation--files)
3. [Text Processing](#text-processing)
4. [Permissions & Ownership](#permissions--ownership)
5. [Users & Groups](#users--groups)
6. [Processes](#processes)
7. [Networking](#networking-commands)
8. [Disk & Storage](#disk--storage)
9. [Package Management](#package-management)
10. [systemd & Services](#systemd--services)
11. [Shell & Scripting](#shell--scripting)
12. [SSH](#ssh)
13. [Performance Analysis](#performance-analysis)
14. [Logs](#logs)
15. [Cron & Scheduling](#cron--scheduling)
16. [Security Essentials](#security-essentials)
17. [Environment Variables](#environment-variables)
18. [Kernel & System Info](#kernel--system-info)

---

## File System

### Directory Structure
```
/               Root
├── bin         Essential user binaries
├── boot        Bootloader files, kernel
├── dev         Device files
├── etc         System configuration files
├── home        User home directories
├── lib         Shared libraries
├── media       Mount points for removable media
├── mnt         Temporary mount points
├── opt         Optional application software
├── proc        Virtual filesystem (process/kernel info)
├── root        Root user's home
├── run         Runtime data (PIDs, sockets)
├── srv         Service data
├── sys         Virtual filesystem (kernel/hardware info)
├── tmp         Temporary files (cleared on reboot)
├── usr         User programs
│   ├── bin     Non-essential user binaries
│   ├── lib     Libraries for /usr/bin
│   ├── local   Locally installed software
│   └── share   Architecture-independent data
└── var         Variable data (logs, databases, mail)
```

---

## Navigation & Files

### Navigation
```bash
pwd                     print working directory
cd ~                    home directory
cd -                    previous directory
ls -la                  list all with details
ls -lah                 human-readable sizes
ls -lt                  sort by time
ls -lS                  sort by size
tree -L 2               tree view, 2 levels deep
```

### File Operations
```bash
cp -r source/ dest/              copy directory recursively
cp -a source/ dest/              copy preserving permissions/timestamps
mv file.txt newname.txt          rename
rm -rf dir/                      remove recursively (CAREFUL)
rm -i file.txt                   prompt before removing
touch file.txt                   create empty file / update timestamp
mkdir -p a/b/c                   create nested dirs
ln -s /path/to/target linkname   symbolic link
ln target linkname               hard link

# Safe alternatives to rm
trash file.txt                   move to trash (install trash-cli)
```

### Finding Files
```bash
find . -name "*.log"             find by name
find . -name "*.log" -mtime -7   modified last 7 days
find . -type f -size +100M       files over 100MB
find . -type f -empty            empty files
find . -perm 777                 find by permissions
find . -user username            find by owner
find . -name "*.py" -exec grep -l "pattern" {} \;   find + grep

locate filename                  fast search (uses database)
updatedb                         update locate's database
which python3                    full path of command
whereis python3                  binary + source + man page
type ls                          show how a command is resolved

# fd — modern find alternative
fd -e py "pattern"               find .py files matching pattern
fd -t f -H "dotfile"             include hidden files
```

### Viewing Files
```bash
cat file.txt
less file.txt                    paginated view (q to quit)
head -20 file.txt                first 20 lines
tail -20 file.txt                last 20 lines
tail -f app.log                  follow (real-time)
tail -f -n 100 app.log           follow last 100 lines
watch -n 2 'cat /proc/meminfo'   refresh command every 2s

# View binary files
hexdump -C file.bin | head
xxd file.bin | head
file filename                    determine file type
strings binary                   extract readable strings from binary
```

### Archives
```bash
# tar
tar -czf archive.tar.gz dir/    compress with gzip
tar -cjf archive.tar.bz2 dir/   compress with bzip2
tar -cJf archive.tar.xz dir/    compress with xz (best ratio)
tar -xzf archive.tar.gz         extract
tar -xzf archive.tar.gz -C /dst extract to directory
tar -tzf archive.tar.gz         list contents

# zip
zip -r archive.zip dir/
unzip archive.zip
unzip -l archive.zip             list contents

# Other
gzip / gunzip file.txt
bzip2 / bunzip2 file.txt
xz / unxz file.txt
```

---

## Text Processing

### grep
```bash
grep "pattern" file.txt
grep -r "pattern" dir/           recursive
grep -r "pattern" --include="*.py" dir/
grep -i "pattern" file           case-insensitive
grep -v "pattern" file           invert (non-matching)
grep -n "pattern" file           show line numbers
grep -c "pattern" file           count matches
grep -l "pattern" dir/           show filenames only
grep -A 3 "pattern" file         3 lines after
grep -B 3 "pattern" file         3 lines before
grep -C 3 "pattern" file         3 lines context
grep -E "regex" file             extended regex
grep -P "perl-regex" file        Perl regex (very powerful)

# Examples
grep -r "TODO" --include="*.py" . | grep -v ".git"
grep -rn "def function_name" .   find function definition
```

### sed
```bash
sed 's/old/new/' file            replace first occurrence per line
sed 's/old/new/g' file           replace all occurrences
sed 's/old/new/gi' file          case-insensitive
sed -i 's/old/new/g' file        edit in-place
sed -i.bak 's/old/new/g' file    in-place with backup
sed -n '10,20p' file             print lines 10–20
sed '10,20d' file                delete lines 10–20
sed '/pattern/d' file            delete matching lines
sed '/^$/d' file                 delete blank lines
sed 's/\s\+$//' file             strip trailing whitespace
```

### awk
```bash
awk '{print $1}' file            print first field
awk '{print $NF}' file           print last field
awk -F: '{print $1}' /etc/passwd  custom delimiter
awk 'NR==5' file                 print 5th line
awk 'NR>=5 && NR<=10' file       print lines 5–10
awk '/pattern/' file             print matching lines
awk '{sum += $1} END {print sum}' file    sum a column
awk '{print NR, $0}' file        add line numbers
awk '!seen[$0]++'                deduplicate lines
awk -F: '{printf "%-20s %s\n", $1, $3}' /etc/passwd   formatted output

# Real-world examples
df -h | awk '$5+0 > 80 {print}'         disks over 80% used
ps aux | awk '{print $1, $4}' | sort -k2 -rn | head  top memory processes
```

### cut, sort, uniq, tr
```bash
cut -d: -f1 /etc/passwd          cut field 1 with : delimiter
cut -c1-10 file                  cut first 10 characters

sort file                        sort alphabetically
sort -n file                     numeric sort
sort -r file                     reverse sort
sort -k2 file                    sort by field 2
sort -t: -k3 -n /etc/passwd      sort by UID

sort file | uniq                 remove duplicates
sort file | uniq -c              count occurrences
sort file | uniq -d              show only duplicates
sort file | uniq -u              show only unique

tr 'a-z' 'A-Z' < file           convert to uppercase
tr -d '\r' < file                remove carriage returns
tr -s ' ' < file                 squeeze repeated spaces

# Power pipelines
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
# → top 10 IP addresses by request count
```

### xargs
```bash
find . -name "*.log" | xargs rm
find . -name "*.py" | xargs grep -l "import os"
echo "a b c" | xargs -n1         one arg per line
cat urls.txt | xargs -P4 -I{} curl -O {}    parallel downloads
find . -name "*.jpg" | xargs -I{} convert {} -quality 85 {}
```

---

## Permissions & Ownership

### Understanding Permissions
```
-rwxr-xr--  1  alice  staff  4096  Jan 1  file.txt
│└──┴──┴──  │   │      │     │     │
│  u  g  o  │  owner  group size  date
│           number of hard links

r = read  (4)
w = write (2)
x = execute (1)

Octal: rwx = 7, rw- = 6, r-x = 5, r-- = 4, -wx = 3, -w- = 2, --x = 1, --- = 0
```

### chmod
```bash
chmod 755 file              rwxr-xr-x
chmod 644 file              rw-r--r-- (common for files)
chmod 600 file              rw------- (private)
chmod +x script.sh          add execute for all
chmod -x file               remove execute for all
chmod u+x file              add execute for owner
chmod g-w file              remove write for group
chmod o=r file              set other to read-only
chmod -R 755 dir/           recursive
```

### chown
```bash
chown alice file            change owner
chown alice:staff file      change owner + group
chown :staff file           change group only
chown -R alice:staff dir/   recursive
```

### Special Permissions
```bash
chmod +s file               setuid/setgid bit
chmod +t /tmp               sticky bit (only owner can delete)
chmod 4755 binary           setuid: runs as file owner
chmod 2755 dir              setgid: new files inherit group
chmod 1777 /tmp             sticky bit: only owner can delete files

# Find setuid files (potential security concern)
find / -perm -4000 -type f 2>/dev/null
```

### umask
```bash
umask                       show current umask (e.g., 0022)
umask 0027                  set umask
# umask 0022 → files created as 644, dirs as 755
# umask 0027 → files created as 640, dirs as 750
```

---

## Users & Groups

```bash
# User management
useradd -m -s /bin/bash username     create user with home + bash
usermod -aG docker username          add to group (requires re-login)
userdel -r username                  delete user + home
passwd username                      set password
chage -l username                    show password aging info

# Current user info
id                          uid, gid, groups
whoami                      just the username
groups                      list groups
w                           who is logged in + what they're doing
last                        recent login history
lastlog                     last login per user

# Switch users
su - username               switch user (full login shell)
sudo -u username command    run command as user
sudo -i                     root login shell
sudo !!                     re-run last command as root

# /etc/passwd format
# username:password:UID:GID:comment:home:shell
cat /etc/passwd | grep -v nologin | grep -v false

# Group management
groupadd groupname
groupmod -n newname oldname
groupdel groupname
gpasswd -a username group   add user to group
gpasswd -d username group   remove user from group
```

---

## Processes

```bash
# Viewing processes
ps aux                       all processes, BSD format
ps -ef                       all processes, System V format
ps aux | grep nginx
pgrep -l nginx               find process by name
pidof nginx                  get PID(s) of process

# Interactive monitoring
top
htop                         better top (if installed)
btop                         much better (if installed)

# Signals
kill PID                     send SIGTERM (graceful)
kill -9 PID                  send SIGKILL (force)
kill -HUP PID                reload config (many daemons)
kill -0 PID                  test if process exists
killall nginx                kill all matching name
pkill -f "python script.py"  kill by full command
```

### Background & Foreground
```bash
command &                    run in background
jobs                         list background jobs
fg                           bring to foreground
fg %2                        bring job 2 to foreground
bg                           resume stopped job in background
Ctrl-Z                       suspend foreground job
nohup command &              run immune to hangup
disown %1                    detach job from shell

# Screen/tmux for persistent sessions
screen -S mysession          new named session
screen -r mysession          reattach
screen -ls                   list sessions
# tmux is preferred — see tmux for more
```

### Process Priority
```bash
nice -n 10 command           start with lower priority (-20 to 19)
renice 10 -p PID             change priority of running process
ionice -c 3 -p PID           set I/O priority to idle
```

### strace & lsof
```bash
strace command               trace system calls
strace -p PID                attach to running process
strace -e trace=network -p PID  only network syscalls

lsof                         list open files
lsof -p PID                  open files for process
lsof -i :80                  processes using port 80
lsof -i TCP:1-1024           all privileged ports in use
lsof +D /var/log             open files in directory
lsof -u alice                files opened by user
```

---

## Networking Commands

```bash
# Interfaces
ip addr show                 show IP addresses
ip addr show eth0            show specific interface
ip link show                 show link status
ip route show                show routing table
ip route get 8.8.8.8         which route would be used

# Legacy commands (still common)
ifconfig                     show interfaces
route -n                     show routing table
netstat -tlnp                listening TCP ports
netstat -tulnp               TCP + UDP listening

# Modern alternatives
ss -tlnp                     listening TCP (fast, modern netstat)
ss -tlnp | grep :80          who's on port 80
ss -s                        socket statistics summary

# DNS
dig google.com               DNS lookup
dig google.com MX            mail exchange records
dig google.com +short        just the IPs
dig @8.8.8.8 google.com      use specific DNS server
nslookup google.com
host google.com

# Connectivity
ping -c 4 google.com
traceroute google.com
mtr google.com               real-time traceroute
curl -v https://example.com  verbose HTTP request
curl -I https://example.com  headers only
curl -o file.txt URL         download to file
wget URL                     download file

# Firewall (see networking guide for more)
ufw status
ufw allow 22
ufw allow 80/tcp
iptables -L -n -v
```

---

## Disk & Storage

```bash
df -h                        disk usage by filesystem
df -i                        inode usage
du -sh dir/                  size of directory
du -sh * | sort -h           sizes of all items, sorted
du -ah --max-depth=1 /var    drill down one level
ncdu                         interactive disk usage (install)

# Blocks & partitions
lsblk                        list block devices (tree view)
lsblk -f                     show filesystems
blkid                        show UUIDs + filesystems
fdisk -l                     list partitions (root)
parted -l                    list partitions (GPT-aware)

# Mount
mount /dev/sdb1 /mnt
mount -t nfs host:/share /mnt
umount /mnt
df -h | grep /mnt
cat /etc/fstab               persistent mounts
mount -a                     mount all in fstab

# Check filesystem
fsck /dev/sdb1               check + repair (unmounted!)
e2fsck -f /dev/sdb1          ext4 specific check

# Swap
swapon --show
free -h
swapoff -a                   disable all swap
swapon -a                    enable all swap from fstab
dd if=/dev/zero of=/swapfile bs=1M count=2048   create swapfile
mkswap /swapfile
swapon /swapfile
```

---

## Package Management

### Debian/Ubuntu (apt)
```bash
apt update                   update package lists
apt upgrade                  upgrade installed packages
apt full-upgrade             upgrade + handle removals
apt install package
apt install -y package       non-interactive
apt remove package           remove (keep config)
apt purge package            remove + config
apt autoremove               remove unused dependencies
apt search keyword
apt show package             package info
apt list --installed
dpkg -l | grep package       check if installed
dpkg -L package              list installed files

# Sources
add-apt-repository ppa:user/ppa
```

### RHEL/CentOS/Fedora (dnf/yum)
```bash
dnf update
dnf install package
dnf remove package
dnf search keyword
dnf info package
dnf list installed
rpm -qa | grep package
rpm -ql package              list installed files
```

### macOS (Homebrew)
```bash
brew install package
brew upgrade package
brew remove package
brew search keyword
brew info package
brew list
brew doctor                  diagnose issues
brew cleanup                 remove old versions
```

---

## systemd & Services

```bash
# Service management
systemctl start service
systemctl stop service
systemctl restart service
systemctl reload service     reload config without restart
systemctl enable service     start on boot
systemctl disable service    don't start on boot
systemctl enable --now service   enable + start immediately
systemctl status service     show status + recent logs
systemctl is-active service
systemctl is-enabled service
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed

# journald logs
journalctl                   all logs
journalctl -u service        logs for service
journalctl -f                follow (like tail -f)
journalctl -n 100            last 100 entries
journalctl --since "2024-01-01"
journalctl --since "1 hour ago"
journalctl -p err -b         errors since last boot
journalctl -b -1             previous boot
journalctl --disk-usage      how much space logs use
journalctl --vacuum-time=7d  keep only last 7 days

# Timers (cron replacement)
systemctl list-timers
systemctl status cron

# System state
systemctl get-default        show default target (runlevel)
systemctl set-default multi-user.target
systemctl rescue             switch to rescue mode
reboot / poweroff / halt
```

### Writing a systemd service
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start
ExecStop=/opt/myapp/bin/stop
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload      after editing service files
```

---

## Shell & Scripting

### Bash Essentials
```bash
# Variables
NAME="Alice"
echo "$NAME"
echo "${NAME}World"          brace expansion
echo "${NAME:-default}"      default if unset
echo "${NAME:=default}"      set default if unset
echo "${#NAME}"              length of variable
echo "${NAME^^}"             uppercase
echo "${NAME,,}"             lowercase
echo "${NAME:0:3}"           substring (start:length)

# Arrays
arr=(one two three)
echo "${arr[0]}"
echo "${arr[@]}"             all elements
echo "${#arr[@]}"            length
arr+=("four")                append

# Arithmetic
echo $((2 + 3))
echo $((10 % 3))
x=$((x + 1))
((x++))

# Command substitution
files=$(ls *.py)
count=$(find . -name "*.log" | wc -l)

# Exit status
command && echo "success" || echo "failed"
if command; then echo "worked"; fi
$?                           exit status of last command
```

### Control Flow
```bash
# if
if [[ -f file.txt ]]; then
    echo "file exists"
elif [[ -d dir ]]; then
    echo "dir exists"
else
    echo "neither"
fi

# Test flags
-f  file exists (and is regular file)
-d  directory exists
-e  exists (any type)
-r  readable
-w  writable
-x  executable
-s  size > 0
-z  string is empty
-n  string is not empty
=   strings equal
!=  strings not equal
-eq -ne -lt -le -gt -ge  numeric comparisons

# for loop
for f in *.txt; do
    echo "Processing: $f"
done

for i in {1..10}; do echo $i; done

for ((i=0; i<10; i++)); do echo $i; done

# while loop
while IFS= read -r line; do
    echo "$line"
done < file.txt

while true; do
    sleep 5
    check_something
done

# case
case "$1" in
    start) start_service ;;
    stop)  stop_service ;;
    *)     echo "Usage: $0 {start|stop}" ;;
esac
```

### Functions
```bash
function greet() {
    local name="$1"              local scope
    local result
    result="Hello, $name!"
    echo "$result"               return via stdout
    return 0                     return exit code
}

output=$(greet "Alice")
greet "World"
```

### Scripting Best Practices
```bash
#!/usr/bin/env bash
set -euo pipefail                # exit on error, undefined vars, pipe fails
IFS=$'\n\t'                      # safer IFS

# Logging
log()    { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >&2; }
info()   { log "INFO:  $*"; }
warn()   { log "WARN:  $*"; }
error()  { log "ERROR: $*"; exit 1; }

# Cleanup on exit
cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT
tmpfile=$(mktemp)

# Usage
usage() {
    echo "Usage: $0 [-h] [-v] [-n name] <arg>"
    echo "  -h  help"
    echo "  -v  verbose"
    echo "  -n  name"
    exit 1
}

# Parse options
verbose=false
name=""
while getopts ":hvn:" opt; do
    case $opt in
        h) usage ;;
        v) verbose=true ;;
        n) name="$OPTARG" ;;
        :) error "Option -$OPTARG requires an argument" ;;
        \?) error "Unknown option: -$OPTARG" ;;
    esac
done
shift $((OPTIND - 1))            remaining args
```

---

## SSH

```bash
ssh user@host
ssh -p 2222 user@host            non-standard port
ssh -i ~/.ssh/id_ed25519 user@host  specify key
ssh -L 8080:localhost:80 user@host  local port forward
ssh -R 8080:localhost:80 user@host  remote port forward
ssh -D 1080 user@host            dynamic SOCKS proxy
ssh -N -f user@host              background tunnel, no shell
ssh -A user@host                 forward SSH agent
ssh -J jump user@final           jump through bastion host

# Key management
ssh-keygen -t ed25519 -C "email" -f ~/.ssh/mykey   generate key
ssh-copy-id user@host            copy public key
ssh-add ~/.ssh/id_ed25519        add key to agent
ssh-add -l                       list keys in agent

# Config file: ~/.ssh/config
Host myserver
    HostName 203.0.113.1
    User alice
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes

Host bastion
    HostName bastion.example.com
    User ec2-user

Host prod-internal
    HostName 10.0.1.5
    User ubuntu
    ProxyJump bastion

# After editing config:
# ssh myserver  (uses all settings above)
# ssh prod-internal  (automatically tunnels through bastion)
```

---

## Performance Analysis

```bash
# CPU
top / htop / btop
vmstat 1                     CPU, memory, swap stats per second
mpstat -P ALL 1              per-CPU stats
perf top                     real-time CPU profiling
sar -u 1 5                   CPU utilization (5 samples, 1s apart)

# Memory
free -h
cat /proc/meminfo
vmstat -s
smem -t -p -k | head -10     memory per process with shared mem

# I/O
iotop                        disk I/O per process
iostat -xz 1                 disk I/O stats
sar -d 1 5                   block device stats

# Network
iftop                        bandwidth per connection
nethogs                      bandwidth per process
nload                        network load overview
ss -s                        socket summary

# System overview
dstat                        combined stats (CPU, disk, net, memory)
glances                      comprehensive monitoring (python)

# Load average
uptime
# Load avg: 0.52, 0.71, 0.89 (1 min, 5 min, 15 min)
# Ideal: < number of CPU cores
nproc                        number of CPU cores

# Identify bottlenecks
# High load, low CPU usage → I/O wait (check iostat)
# High load, high CPU usage → CPU-bound (check top)
# High load, low both → memory/swap pressure (check free, vmstat)
```

---

## Logs

```bash
# Traditional log files
/var/log/syslog              general system logs (Debian)
/var/log/messages            general system logs (RHEL)
/var/log/auth.log            authentication logs
/var/log/nginx/              Nginx logs
/var/log/apache2/            Apache logs
/var/log/mysql/              MySQL logs

# Live viewing
tail -f /var/log/syslog
multitail /var/log/syslog /var/log/auth.log    watch multiple

# Searching
grep "ERROR" /var/log/syslog | tail -50
grep -E "error|warn|crit" /var/log/syslog -i
zgrep "error" /var/log/syslog.*.gz             search rotated logs

# Log rotation
cat /etc/logrotate.conf
logrotate -f /etc/logrotate.conf               force rotation
```

---

## Cron & Scheduling

```bash
crontab -e                   edit user crontab
crontab -l                   list user crontab
crontab -r                   remove user crontab
sudo crontab -u alice -l     view another user's crontab

# Cron syntax
# ┌─────────── minute (0-59)
# │ ┌───────── hour (0-23)
# │ │ ┌─────── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌─── day of week (0-7, 0=7=Sun)
# │ │ │ │ │
# * * * * *  command

# Examples
0 * * * *      command          every hour at :00
*/15 * * * *   command          every 15 minutes
0 2 * * *      command          daily at 2am
0 2 * * 0      command          weekly, Sundays at 2am
0 2 1 * *      command          monthly, 1st at 2am
@reboot        command          on system startup
@daily         command          once a day (midnight)

# Best practice: always use full paths in cron
# And redirect output to log:
0 2 * * * /usr/bin/backup.sh >> /var/log/backup.log 2>&1
```

---

## Security Essentials

```bash
# Check running services
ss -tlnp
systemctl list-units --type=service --state=running

# Find setuid binaries
find / -perm -4000 -type f 2>/dev/null

# Check sudoers
cat /etc/sudoers
sudo -l                      what can current user sudo

# Check auth logs for failed logins
grep "Failed password" /var/log/auth.log | tail -20
grep "Invalid user" /var/log/auth.log | awk '{print $8}' | sort | uniq -c | sort -rn

# Firewall
ufw status verbose
iptables -L -n -v --line-numbers

# Open ports vs expected
ss -tlnp | awk '{print $4}' | cut -d: -f2 | sort -n | uniq

# File integrity
md5sum file1 file2            generate checksums
sha256sum file                preferred hash
sha256sum -c checksums.txt    verify checksums

# auditd — file access auditing
auditctl -w /etc/passwd -p rwxa -k passwd_changes
ausearch -k passwd_changes

# Last logins + actions
last | head -20
lastlog
history | grep -E "curl|wget|chmod|chown|rm -rf" | head -20
```

---

## Environment Variables

```bash
env                          show all environment variables
printenv PATH                show specific variable
echo $HOME

# Setting
export VAR=value             set in current shell + children
VAR=value command            set only for that command

# Making persistent:
# bash: ~/.bashrc or ~/.bash_profile or ~/.profile
# zsh: ~/.zshrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc             reload (or `. ~/.bashrc`)

# System-wide: /etc/environment, /etc/profile.d/*.sh

# Useful variables
$HOME           home directory
$PATH           command search path
$SHELL          current shell
$USER           username
$PWD            current directory
$OLDPWD         previous directory
$EDITOR         default editor
$LANG           locale
$TERM           terminal type
$?              last exit code
$$              PID of current shell
$!              PID of last background job
$0              script name
$1..$9          positional parameters
$@              all arguments (as separate words)
$#              number of arguments
```

---

## Kernel & System Info

```bash
uname -r                     kernel version
uname -a                     all kernel info
cat /etc/os-release          distribution info
lsb_release -a               distribution info (if installed)
hostname -f                  FQDN
hostnamectl                  system hostname info

# Hardware
lscpu                        CPU details
lsmem                        memory configuration
dmidecode -t memory          memory slots (root)
lspci                        PCI devices
lspci -v                     verbose
lsusb                        USB devices
lshw -short                  all hardware (root)
hwinfo --short               hardware info

# Kernel modules
lsmod                        loaded modules
modinfo module               module details
modprobe module              load module
rmmod module                 remove module

# Limits
ulimit -a                    show all limits
ulimit -n 65535              set open files limit
cat /proc/sys/fs/file-max    system-wide file limit
sysctl -a | grep net         network settings
sysctl -w net.ipv4.ip_forward=1   set temporarily
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && sysctl -p  persist
```
