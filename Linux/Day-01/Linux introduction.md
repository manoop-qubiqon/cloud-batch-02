# Linux Introduction

## What is Linux?

Linux is a free, open-source operating system kernel first created by **Linus Torvalds** in 1991. Combined with tools from the GNU Project, it forms a complete OS used across servers, desktops, embedded systems, mobile devices (Android), and cloud infrastructure.

Unlike proprietary systems like Windows or macOS, Linux is distributed under the **GNU General Public License (GPL)** — meaning anyone can view, modify, and redistribute the source code.

---

## Linux Architecture

```
┌─────────────────────────────┐
│        User Applications    │  ← Programs you run (bash, vim, python...)
├─────────────────────────────┤
│         Shell / CLI         │  ← Interface between user and kernel (bash, zsh)
├─────────────────────────────┤
│           Kernel            │  ← Core of the OS (process, memory, device mgmt)
├─────────────────────────────┤
│           Hardware          │  ← CPU, RAM, Disk, Network interfaces
└─────────────────────────────┘
```

### Key Components

| Component | Role |
|-----------|------|
| **Kernel** | Manages hardware resources, processes, memory |
| **Shell** | Command-line interpreter (bash, zsh, sh) |
| **File System** | Organizes data on disk (ext4, xfs, btrfs) |
| **Package Manager** | Installs/removes software (apt, yum, dnf, pacman) |
| **Init System** | First process to run on boot (systemd, SysVinit) |

---

## Popular Linux Distributions

A **distribution (distro)** bundles the Linux kernel with software, tools, and a package manager.

| Distro | Base | Package Manager | Best For |
|--------|------|----------------|----------|
| Ubuntu | Debian | `apt` | Beginners, desktops, servers |
| Debian | — | `apt` | Stability, servers |
| CentOS / RHEL | Red Hat | `yum` / `dnf` | Enterprise servers |
| Fedora | Red Hat | `dnf` | Developers, cutting-edge |
| Arch Linux | — | `pacman` | Advanced users, customization |
| Kali Linux | Debian | `apt` | Penetration testing, security |
| Alpine Linux | — | `apk` | Docker containers, minimal footprint |

---

## Linux File System Hierarchy

Everything in Linux is a file. The file system follows the **Filesystem Hierarchy Standard (FHS)**.

```
/                   Root of the entire file system
├── bin/            Essential user binaries (ls, cp, mv)
├── boot/           Boot loader files, kernel images
├── dev/            Device files (disk, terminal, null)
├── etc/            System-wide configuration files
├── home/           User home directories (/home/manoop)
├── lib/            Shared libraries for /bin and /sbin
├── media/          Mount points for removable media
├── mnt/            Temporary mount points
├── opt/            Optional/third-party software
├── proc/           Virtual FS for process/kernel info
├── root/           Home directory for root user
├── sbin/           System binaries (only for root)
├── sys/            Virtual FS exposing kernel objects
├── tmp/            Temporary files (cleared on reboot)
├── usr/            User programs, libraries, docs
└── var/            Variable data: logs, caches, spool
```

---

## Essential Linux Commands

### Navigation & File Operations

```bash
pwd                     # Print working directory
ls -la                  # List files (long format, hidden files)
cd /path/to/dir         # Change directory
cd ~                    # Go to home directory
mkdir -p dir/subdir     # Create nested directories
rm -rf dir/             # Remove directory recursively (use with caution!)
cp -r src/ dest/        # Copy directory
mv oldname newname      # Move or rename
find / -name "*.log"    # Find files by name
```

### Viewing & Editing Files

```bash
cat file.txt            # Display file contents
less file.txt           # Scrollable view
head -n 20 file.txt     # First 20 lines
tail -f /var/log/syslog # Follow live log output
nano file.txt           # Simple text editor
vim file.txt            # Advanced editor (i=insert, :wq=save&quit, :q!=quit)
```

### File Permissions

```bash
ls -l file.txt          # View permissions: -rwxr-xr--
chmod 755 script.sh     # rwxr-xr-x (owner=rwx, group=rx, others=rx)
chmod +x script.sh      # Add execute permission
chown user:group file   # Change owner and group
```

Permission breakdown:

```
- rwx r-x r--
│ │   │   └── Others: read only
│ │   └────── Group: read + execute
│ └────────── Owner: read + write + execute
└──────────── File type (- = file, d = dir, l = link)
```

### Process Management

```bash
ps aux                  # List all running processes
top                     # Live process viewer
htop                    # Enhanced process viewer (install separately)
kill -9 PID             # Force kill process by ID
pkill nginx             # Kill process by name
nohup command &         # Run process in background, immune to hangups
jobs                    # List background jobs
```

### Networking

```bash
ip a                    # Show network interfaces and IPs
ping google.com         # Test connectivity
curl -I https://site.com  # HTTP request headers
wget https://site.com/file  # Download file
ss -tuln                # Show listening ports
netstat -tulnp          # Similar to ss (older tool)
ssh user@host           # Connect to remote machine
scp file.txt user@host:/path  # Secure copy to remote
```

### Package Management (Ubuntu/Debian)

```bash
sudo apt update         # Refresh package index
sudo apt upgrade        # Upgrade installed packages
sudo apt install nginx  # Install a package
sudo apt remove nginx   # Remove a package
sudo apt autoremove     # Clean up unused dependencies
dpkg -l | grep nginx    # List installed packages matching nginx
```

### Disk & Storage

```bash
df -h                   # Disk usage (human-readable)
du -sh /var/log/        # Size of a directory
lsblk                   # List block devices
mount /dev/sdb1 /mnt    # Mount a disk
umount /mnt             # Unmount a disk
```

### User Management

```bash
whoami                  # Current logged-in user
id                      # User ID, group ID
sudo command            # Run command as root
su - username           # Switch to another user
useradd -m manoop       # Create a new user with home dir
passwd manoop           # Set/change user password
usermod -aG docker manoop  # Add user to a group
cat /etc/passwd         # List all system users
```

---

## Shell & Scripting Basics

### Environment Variables

```bash
echo $HOME              # Print home directory
echo $PATH              # Print executable search path
export MY_VAR="hello"   # Set an environment variable
env                     # List all environment variables
```

### Redirection & Pipes

```bash
command > file.txt      # Redirect stdout to file (overwrite)
command >> file.txt     # Append stdout to file
command 2> error.log    # Redirect stderr
command &> all.log      # Redirect both stdout and stderr
command1 | command2     # Pipe output of cmd1 to cmd2

# Examples
ls -la | grep ".sh"
cat access.log | grep "404" | wc -l
```

### Basic Shell Script

```bash
#!/bin/bash
# my_script.sh

NAME="Manoop"
echo "Hello, $NAME!"

for i in 1 2 3; do
  echo "Iteration $i"
done

if [ -f "/etc/hosts" ]; then
  echo "Hosts file exists"
fi
```

```bash
chmod +x my_script.sh
./my_script.sh
```

---

## systemd — Service Management

Most modern Linux distros use **systemd** as the init system.

```bash
sudo systemctl start nginx       # Start a service
sudo systemctl stop nginx        # Stop a service
sudo systemctl restart nginx     # Restart a service
sudo systemctl status nginx      # Check service status
sudo systemctl enable nginx      # Enable on boot
sudo systemctl disable nginx     # Disable on boot
journalctl -u nginx -f           # Live logs for a service
journalctl -xe                   # System error logs
```

---

## Linux vs Windows vs macOS

| Feature | Linux | Windows | macOS |
|---------|-------|---------|-------|
| Cost | Free | Paid | Free (with Apple hardware) |
| Source | Open-source | Closed-source | Partially open |
| Shell | bash/zsh (native) | PowerShell/CMD | zsh (native) |
| Package Manager | apt/yum/pacman | winget (limited) | Homebrew (3rd party) |
| Server Market | Dominant (~96%) | Minority | Rare |
| Customization | Very high | Limited | Moderate |
| Security | High | Moderate | High |

---

## Why Linux Matters for Cloud & DevOps

- **Cloud VMs** — AWS EC2, Azure VMs, and GCP Compute instances run Linux by default
- **Containers** — Docker and Kubernetes are built on Linux kernel features (cgroups, namespaces)
- **CI/CD Pipelines** — Azure Pipelines, GitHub Actions runners are Linux-based
- **Infrastructure as Code** — Terraform, Ansible, and shell scripts target Linux environments
- **Cost** — No licensing fees; scales cheaply on cloud

> **Key Insight:** A solid grip on Linux is foundational for any cloud data engineering or DevOps role. Every Azure Ubuntu VM, every container, every pipeline agent runs Linux under the hood.

---

## Useful Shortcuts & Tips

| Shortcut | Action |
|----------|--------|
| `Ctrl + C` | Kill running process |
| `Ctrl + Z` | Suspend process (resume with `fg`) |
| `Ctrl + D` | Logout / close terminal |
| `Ctrl + L` | Clear terminal screen |
| `Ctrl + R` | Reverse search command history |
| `Tab` | Auto-complete file/command names |
| `!!` | Repeat last command |
| `!$` | Last argument of previous command |
| `man command` | Open manual page for any command |
| `command --help` | Quick help for a command |

---

## Quick Reference Card

```
Navigation    : pwd, ls, cd, find
Files         : cat, less, head, tail, nano, vim, cp, mv, rm
Permissions   : chmod, chown, ls -l
Processes     : ps, top, kill, nohup
Network       : ip, ping, curl, ssh, scp, ss
Packages      : apt update/install/remove
Disk          : df, du, lsblk, mount
Users         : whoami, sudo, useradd, passwd
Services      : systemctl start/stop/enable/status
Logs          : journalctl -u <service> -f
```

---

*Last updated: June 2026*