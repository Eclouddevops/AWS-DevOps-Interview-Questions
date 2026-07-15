# Linux & Windows Servers — Deep-Dive Interview Q&A

## Table of Contents
1. [Linux Fundamentals](#linux-fundamentals)
2. [Linux Troubleshooting](#linux-troubleshooting)
3. [Linux Networking](#linux-networking)
4. [Windows Server](#windows-server)
5. [Tricky Scenarios](#tricky-scenarios)

---

## Linux Fundamentals


**Q1: Explain the Linux boot process from BIOS to login prompt.**

**A:**

```
1. BIOS/UEFI → POST (hardware check) → finds boot device
2. Bootloader (GRUB2) → loads kernel + initramfs
3. Kernel → initializes hardware, mounts root filesystem
4. Init system (systemd) → PID 1, starts services
5. Target/Runlevel → multi-user.target or graphical.target
6. Login prompt (getty/sshd)
```

**systemd targets (replacing runlevels):**
| Target | Equivalent | Description |
|--------|-----------|-------------|
| poweroff.target | 0 | Shutdown |
| rescue.target | 1 | Single user (recovery) |
| multi-user.target | 3 | Multi-user, no GUI |
| graphical.target | 5 | Multi-user with GUI |
| reboot.target | 6 | Reboot |

```bash
# Check current target
systemctl get-default

# Change default target
systemctl set-default multi-user.target

# Check boot time breakdown
systemd-analyze blame  # Time per service
systemd-analyze critical-chain  # Critical path
```

**Tricky**: If a server won't boot, use `rescue.target` (single user mode) via GRUB menu. If that fails, boot from live USB and mount the root filesystem to fix.

---

**Q2: Explain Linux file permissions, SUID, SGID, and Sticky Bit.**

**A:**

```
-rwxr-xr-x  1 root root  4096 Jan 1 /usr/bin/app
│└─┬─┘└┬┘└┬┘
│  │   │  │
│  │   │  └── Others: read + execute
│  │   └───── Group: read + execute
│  └───────── Owner: read + write + execute
└──────────── File type (- = file, d = dir, l = symlink)

Numeric: rwx = 4+2+1 = 7
chmod 755 = rwxr-xr-x
chmod 644 = rw-r--r--
```

**Special permissions:**
| Permission | On File | On Directory |
|-----------|---------|-------------|
| SUID (4000) | Execute as file owner | No effect |
| SGID (2000) | Execute as group owner | New files inherit directory's group |
| Sticky (1000) | No effect | Only owner can delete their files |

```bash
# SUID: /usr/bin/passwd runs as root even when normal user executes it
-rwsr-xr-x root root /usr/bin/passwd  # 's' in owner execute = SUID

# SGID on directory: Team collaboration
chmod 2775 /shared/project  # New files get group "project"

# Sticky bit: /tmp — users can create files but can't delete others' files
drwxrwxrwt root root /tmp  # 't' at end = sticky bit
```

**Tricky security question**: Find all SUID binaries (potential privilege escalation):
```bash
find / -perm -4000 -type f 2>/dev/null
# Attackers look for SUID binaries they can exploit
```

---

**Q3: Explain Linux process management — signals, zombies, orphans.**

**A:**

**Key signals:**
| Signal | Number | Default | Use |
|--------|--------|---------|-----|
| SIGHUP | 1 | Terminate | Reload config (daemons) |
| SIGINT | 2 | Terminate | Ctrl+C |
| SIGKILL | 9 | Kill (can't catch) | Force kill |
| SIGTERM | 15 | Terminate | Graceful shutdown (default kill) |
| SIGSTOP | 19 | Stop (can't catch) | Pause process |
| SIGCONT | 18 | Continue | Resume paused process |

**Zombie processes:**
```
Parent forks child → Child finishes → Sends SIGCHLD to parent
→ Parent calls wait() → Child fully cleaned up

If parent DOESN'T call wait():
→ Child becomes ZOMBIE (Z state in ps)
→ Entry stays in process table (wastes PID)
→ Can't be killed (already dead!)
```

Fix: Kill the PARENT (zombies are cleaned by init/systemd adopting them).

```bash
# Find zombies
ps aux | grep 'Z'

# Find zombie's parent
ps -o ppid= -p <zombie_pid>

# Kill parent to clean zombies
kill <parent_pid>
```

**Orphan processes:** Parent dies → orphan adopted by PID 1 (systemd) → cleaned up properly.

---

## Linux Troubleshooting

**Q4: Server is slow. Walk through the complete diagnostic process.**

**A:**

```bash
# 1. Overview — what's the load?
uptime
# 15:00 up 30 days, load average: 8.50, 6.20, 3.10
# Load > number of CPUs = overloaded
# Rising load (3.10 → 6.20 → 8.50) = getting worse

# 2. CPU — who's using it?
top -bn1 | head -20
# Look at: %Cpu(s): us (user), sy (system), wa (I/O wait), id (idle)
# High wa = disk bottleneck
# High us = application busy
# High sy = kernel/system calls busy

# 3. Memory — is swap being used?
free -h
# If swap usage is high → memory pressure → swapping = SLOW
vmstat 1 5
# si/so columns = swap in/out (should be 0)

# 4. Disk I/O — is disk the bottleneck?
iostat -x 1 3
# %util near 100% = disk saturated
# await > 10ms = I/O latency issue
iotop  # Which process is doing I/O?

# 5. Network — is network saturated?
sar -n DEV 1 3  # Network interface stats
ss -s  # Socket summary (total connections)
ss -tlnp  # What's listening?

# 6. Processes — what's actually running?
ps aux --sort=-%cpu | head -10  # Top CPU consumers
ps aux --sort=-%mem | head -10  # Top memory consumers

# 7. Disk space
df -h  # Filesystem usage
df -i  # Inode usage (can be full even with space available!)
```

---

**Q5: Disk is 100% full but you can't find what's using space. What's happening?**

**A:**

**Common causes:**

1. **Deleted files still held open by processes:**
```bash
# Files deleted but process still has handle open
lsof +L1  # Shows deleted files still open
# Often: large log files deleted but app still writing to them
# Fix: Restart the process OR truncate: > /proc/<pid>/fd/<fd_number>
```

2. **Inode exhaustion (no space but df shows free):**
```bash
df -i  # Check inode usage
# Millions of tiny files (session files, temp files)
find /tmp -type f | wc -l
```

3. **Hidden mount points:**
```bash
# Files written to directory BEFORE something was mounted there
# Original files hidden under mount, still using space
umount /data  # Reveals hidden files underneath
```

4. **Large files in unexpected places:**
```bash
du -sh /* 2>/dev/null | sort -rh | head -10
find / -xdev -type f -size +100M 2>/dev/null
# Check /var/log, /tmp, /var/spool
```

5. **Journal logs:**
```bash
journalctl --disk-usage  # Can grow to GB
journalctl --vacuum-size=500M  # Trim to 500MB
```

---

**Q6: Explain systemd service management. How do you create and troubleshoot services?**

**A:**

```bash
# Service management
systemctl start/stop/restart/reload nginx
systemctl enable/disable nginx  # Start on boot
systemctl status nginx  # Current status + recent logs
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --failed  # All failed services

# View service logs
journalctl -u nginx -f  # Follow logs
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err  # Only errors
```

**Custom service file** (`/etc/systemd/system/myapp.service`):
```ini
[Unit]
Description=My Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=notify
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
LimitNOFILE=65535
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
# After creating/modifying service file:
systemctl daemon-reload
systemctl enable --now myapp
```

**Tricky**: `Type=notify` means systemd waits for the app to signal readiness (via sd_notify). If the app doesn't support it, use `Type=simple` (assumes ready immediately) or `Type=forking` (for traditional daemons that fork).

---

## Linux Networking

**Q7: Explain iptables/nftables. How does Linux firewall work?**

**A:**

```
Packet flow through iptables chains:

INCOMING: PREROUTING → INPUT → Application
FORWARDED: PREROUTING → FORWARD → POSTROUTING
OUTGOING: Application → OUTPUT → POSTROUTING

Tables (in processing order):
├── raw: Connection tracking exceptions
├── mangle: Packet header modification
├── nat: Network Address Translation
├── filter: Accept/Drop/Reject (DEFAULT)
└── security: SELinux/MAC rules
```

```bash
# View rules
iptables -L -n -v  # Filter table (default)
iptables -t nat -L -n -v  # NAT table

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow established connections (CRITICAL for stateful filtering)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block everything else
iptables -A INPUT -j DROP

# NAT/Masquerade (for routing through this host)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Save rules (persist across reboot)
iptables-save > /etc/iptables/rules.v4
```

**Tricky**: Order matters! Rules are evaluated top-to-bottom, first match wins. If you add DROP before ACCEPT, all traffic is dropped. Always put ACCEPT rules before DROP.

---

**Q8: A server can't reach the internet. How do you troubleshoot layer by layer?**

**A:**

```bash
# Layer 1: Interface up?
ip link show
ip addr show  # Has IP?

# Layer 2: Local connectivity?
ping -c 1 <gateway_ip>  # Can reach gateway?
ip route show  # Default route exists?
# If no route: ip route add default via <gateway_ip>

# Layer 3: Can reach external IPs?
ping -c 1 8.8.8.8  # Bypass DNS
traceroute 8.8.8.8  # Where does it stop?

# Layer 4: DNS working?
nslookup google.com  # DNS resolution
cat /etc/resolv.conf  # DNS config correct?
dig google.com @8.8.8.8  # Test with known DNS

# Layer 5: Application connectivity?
curl -v https://google.com  # Full HTTP test
curl -x "" https://google.com  # Bypass proxy

# Common fixes:
# No default route:
ip route add default via 10.0.0.1

# DNS not resolving:
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# Firewall blocking:
iptables -L OUTPUT -n  # Check outbound rules
```

---

**Q9: Explain SSH key management, tunneling, and hardening.**

**A:**

```bash
# Key generation (Ed25519 preferred over RSA)
ssh-keygen -t ed25519 -C "user@company.com"

# SSH tunneling (port forwarding):
# Local forward: Access remote service via local port
ssh -L 8080:remote-db:3306 bastion-host
# Now: localhost:8080 → bastion → remote-db:3306

# Remote forward: Expose local service to remote
ssh -R 9090:localhost:3000 remote-server
# Now: remote-server:9090 → your-machine:3000

# Dynamic SOCKS proxy (browse through SSH)
ssh -D 1080 bastion-host
# Configure browser to use localhost:1080 as SOCKS proxy

# SSH config (~/.ssh/config):
Host bastion
    HostName 10.0.1.5
    User ec2-user
    IdentityFile ~/.ssh/prod.pem

Host private-server
    HostName 10.0.2.10
    User ubuntu
    ProxyJump bastion  # Jump through bastion automatically
```

**SSH hardening** (`/etc/ssh/sshd_config`):
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers deploy admin
Protocol 2
Port 2222  # Non-standard port (security through obscurity, minor benefit)
```

---

## Windows Server

**Q10: Essential Windows Server commands for DevOps engineers.**

**A:**

```powershell
# System Information
Get-ComputerInfo
systeminfo

# Services
Get-Service | Where-Object {$_.Status -eq 'Running'}
Start-Service -Name "W3SVC"  # IIS
Restart-Service -Name "W3SVC" -Force
New-Service -Name "MyApp" -BinaryPathName "C:\app\myapp.exe" -StartupType Automatic

# Processes
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Stop-Process -Name "hung-app" -Force

# Network
Get-NetIPConfiguration
Test-NetConnection -ComputerName server01 -Port 443
Get-NetTCPConnection -State Listen
New-NetFirewallRule -DisplayName "Allow 8080" -Direction Inbound -Port 8080 -Protocol TCP -Action Allow

# Event Logs
Get-EventLog -LogName Application -Newest 20 -EntryType Error
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 10

# Disk
Get-PSDrive -PSProvider FileSystem
Get-WmiObject Win32_LogicalDisk | Select-Object DeviceID, @{N='FreeGB';E={[math]::Round($_.FreeSpace/1GB,2)}}

# Windows Update
Get-WindowsUpdate  # Requires PSWindowsUpdate module
Install-WindowsUpdate -AcceptAll -AutoReboot

# IIS Management
Import-Module WebAdministration
Get-Website
Start-Website -Name "Default Web Site"
New-WebBinding -Name "MyApp" -IPAddress "*" -Port 443 -Protocol https
```

---

**Q11: How do you troubleshoot a Windows service that won't start?**

**A:**

```powershell
# 1. Check service status and dependencies
Get-Service MyService | Select-Object *
Get-Service MyService | Select-Object -ExpandProperty DependentServices

# 2. Check Event Viewer
Get-WinEvent -FilterHashtable @{
    LogName='Application'
    ProviderName='MyService'
    Level=2  # Error
} -MaxEvents 5

# 3. Try starting manually and capture error
Start-Service MyService -ErrorAction Stop

# 4. Check service account permissions
# Services → Properties → Log On tab
# Verify account has "Log on as a service" right

# 5. Check file permissions
icacls "C:\Program Files\MyService\myservice.exe"

# 6. Check dependencies are running
Get-Service | Where-Object {$_.DependentServices -contains 'MyService'}

# 7. Check port conflicts
netstat -an | findstr ":8080"

# 8. Run as console app (if possible) to see stdout errors
& "C:\Program Files\MyService\myservice.exe" --console
```

**Common causes:**
- Incorrect service account password (changed but not updated)
- Port already in use by another service
- Missing DLL or dependency
- Insufficient permissions on log directory
- Database connection failure during startup
- Certificate expired (for HTTPS services)

---

**Q12: Compare Linux and Windows for server workloads in DevOps context.**

**A:**

| Aspect | Linux | Windows |
|--------|-------|---------|
| Container support | Native (cgroups, namespaces) | Windows Containers (limited) |
| Automation | Bash, Python, Ansible (native) | PowerShell, Ansible (WinRM) |
| Cost | Free (OS), lower EC2 cost | License cost, higher EC2 cost |
| CI/CD agents | Native Docker, lightweight | Heavier, Windows containers slow |
| Security patching | Package managers (apt/yum) | Windows Update, WSUS |
| Configuration mgmt | Ansible, Chef, Puppet (native SSH) | Ansible (WinRM), DSC, Chef |
| Monitoring | Prometheus node_exporter | Windows Exporter, SCOM |
| Remote access | SSH (fast, scriptable) | RDP (GUI), WinRM (PowerShell) |
| File systems | ext4, xfs, btrfs | NTFS, ReFS |
| Package mgmt | apt, yum, dnf | Chocolatey, winget, MSI |

**When to use Windows:**
- .NET Framework applications (not .NET Core)
- SQL Server (native, better performance)
- Active Directory domain services
- Legacy applications requiring Windows APIs
- SharePoint, Exchange

**When to use Linux:**
- Everything else (containers, microservices, databases, web servers)
- Cost-sensitive workloads
- High-performance computing
- DevOps tooling (Jenkins, GitLab runners, build agents)

---

## Tricky Scenarios

**Q13: Server has high load average but CPU usage shows low. What's happening?**

**A:**

**Load average includes processes in D state (uninterruptible sleep = waiting for I/O)!**

```bash
# Check CPU vs I/O wait
top  # Look at %wa (I/O wait)
# If %wa is high → processes waiting for disk

# Check for D-state processes
ps aux | awk '$8 ~ /D/ {print}'

# Check disk I/O
iostat -x 1
# %util = 100% → disk is the bottleneck
# await > 20ms → slow disk responses

# Check what's doing I/O
iotop -o  # Only show processes doing I/O

# Common causes:
# 1. Swap thrashing (out of RAM → swapping to disk)
free -h  # Check swap usage
vmstat 1  # si/so columns

# 2. NFS mount issues (network filesystem hanging)
mount | grep nfs
# NFS server down → processes hang in D state

# 3. Failing disk
dmesg | grep -i "error\|fault\|bad"
smartctl -a /dev/sda
```

**Tricky**: You CANNOT kill D-state processes (not even with SIGKILL). They're waiting for kernel I/O to complete. Fix the underlying I/O issue (fix disk, unmount hung NFS, add RAM to stop swapping).

---

**Q14: How do you handle "Too many open files" errors?**

**A:**

```bash
# Check current limits for a process
cat /proc/<pid>/limits | grep "Max open files"

# Check how many files a process has open
ls /proc/<pid>/fd | wc -l
# Or: lsof -p <pid> | wc -l

# Check system-wide limit
cat /proc/sys/fs/file-nr
# Output: <allocated> <free> <max>

# Increase per-process limit (temporary, current session):
ulimit -n 65535

# Permanent — for a specific service (/etc/systemd/system/myapp.service):
[Service]
LimitNOFILE=65535

# Permanent — system-wide (/etc/security/limits.conf):
*    soft    nofile    65535
*    hard    nofile    65535

# Permanent — sysctl (system max):
echo "fs.file-max = 2097152" >> /etc/sysctl.conf
sysctl -p
```

**Common services that hit this limit:**
- Nginx (one fd per connection × 2 for proxy)
- Java applications (file handles for everything)
- Database servers (connections = file descriptors)
- Container runtimes (Docker daemon)

**Tricky**: The limit applies per-process. A user limit of 1024 means EACH process by that user can open 1024 files, not total. But the system-wide `fs.file-max` IS a total across all processes.

---

**Q15: Explain Linux memory management — buffers, cache, OOM killer.**

**A:**

```bash
$ free -h
              total    used    free    shared  buff/cache  available
Mem:           16G     6G     1G      500M    9G          9G
Swap:          4G      0B     4G

# "free" is misleading! 
# buff/cache = memory used for disk caching (AVAILABLE for applications)
# "available" = what apps can actually use = free + reclaimable cache
```

**Key concepts:**
- **Buffers**: Metadata cache (directory entries, file metadata)
- **Cache**: File content cache (pages read from disk)
- **Available**: Free + easily reclaimable cache (what matters!)
- **Swap**: Overflow to disk when RAM is full

**OOM Killer** (Out-Of-Memory):
```bash
# When system runs out of memory:
# 1. Kernel calculates "oom_score" for each process (0-1000)
# 2. Kills process with highest score
# 3. Higher score = more memory used + less important

# Check OOM score
cat /proc/<pid>/oom_score

# Protect critical processes from OOM (lower = safer)
echo -1000 > /proc/<pid>/oom_score_adj  # Never kill this process
echo 1000 > /proc/<pid>/oom_score_adj   # Kill this first

# Check if OOM killed something
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i "oom"
```

**Tricky**: A server with 1GB "free" and 14GB "buff/cache" is NOT out of memory! Linux aggressively uses RAM for disk caching. This is GOOD (faster disk access). It gives the cache back when apps need it. Only worry if "available" is low.
