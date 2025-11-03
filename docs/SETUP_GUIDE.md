# Complete Setup Guide

This guide provides step-by-step instructions for deploying the entire infrastructure.

## Prerequisites

### Hardware Requirements
- **Host Machine**: 8GB RAM minimum, 16GB recommended
- **Disk Space**: 50GB free space
- **CPU**: 4+ cores recommended

### Software Requirements
- VirtualBox 6.x or later
- Rocky Linux 9 ISO image
- SSH client (OpenSSH)

## Phase 1: Virtual Machine Creation

### 1.1 Create 5 Virtual Machines

For each VM, configure:

**General Settings**:
- OS Type: Linux / Red Hat (64-bit)
- Base Memory: 1024MB (2048MB for DB-central01)
- Disk: 20GB dynamically allocated

**Network Adapter 1** (NAT):
```
Adapter Type: Intel PRO/1000 MT Desktop
Attached to: NAT
```

**Network Adapter 2** (Host-Only):
```
Adapter Type: Intel PRO/1000 MT Desktop
Attached to: Host-Only Adapter
```

### 1.2 VM Details

| Hostname | RAM | IP Address | Purpose |
|----------|-----|------------|---------|
| lb01 | 1GB | 192.168.56.11 | Load Balancer + DNS |
| web01 | 1GB | 192.168.56.21 | Web Server 1 |
| web02 | 1GB | 192.168.56.22 | Web Server 2 |
| storage01 | 1GB | 192.168.56.30 | NFS Storage |
| db-central01 | 2GB | 192.168.56.40 | MariaDB Database |

### 1.3 Install Rocky Linux 9

For each VM:

1. Boot from Rocky Linux 9 ISO
2. Select "Install Rocky Linux 9"
3. **Installation Destination**: Select disk, automatic partitioning
4. **Network & Hostname**:
   - Enable both network adapters
   - Set static IP for Host-Only adapter (enp0s8)
   - Set hostname (lb01, web01, etc.)
5. **Root Password**: Set a strong password
6. **User Creation**: Create 'ansible' user with sudo privileges
7. Begin Installation
8. Reboot after installation

### 1.4 Configure Static IP (each VM)

Edit `/etc/sysconfig/network-scripts/ifcfg-enp0s8`:

```bash
TYPE=Ethernet
BOOTPROTO=none
NAME=enp0s8
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.56.XX  # Replace XX with appropriate IP
PREFIX=24
DNS1=8.8.8.8
```

Restart network:
```bash
sudo nmcli connection reload
sudo nmcli connection up enp0s8
```

Verify:
```bash
ip addr show enp0s8
ping -c 3 8.8.8.8
```

## Phase 2: Ansible Control Node Setup

### 2.1 Install Ansible on LB01

```bash
# SSH into LB01
ssh root@192.168.56.11

# Install EPEL repository
sudo dnf install -y epel-release

# Install Ansible
sudo dnf install -y ansible

# Verify installation
ansible --version
```

Expected output:
```
ansible [core 2.14.x]
  python version = 3.9.x
```

### 2.2 Configure SSH Key Authentication

Generate SSH key on LB01:
```bash
ssh-keygen -t rsa -b 4096 -C "ansible@lb01"
# Press Enter for all prompts (no passphrase)
```

Copy SSH key to all managed nodes:
```bash
ssh-copy-id root@192.168.56.21  # WEB01
ssh-copy-id root@192.168.56.22  # WEB02
ssh-copy-id root@192.168.56.30  # Storage01
ssh-copy-id root@192.168.56.40  # DB-central01
```

Test SSH connectivity:
```bash
ssh root@192.168.56.21 'hostname'
ssh root@192.168.56.22 'hostname'
ssh root@192.168.56.30 'hostname'
ssh root@192.168.56.40 'hostname'
```

### 2.3 Clone Ansible Project

```bash
# Install git
sudo dnf install -y git

# Clone repository
cd /opt
git clone https://github.com/qkrtpdlr/ansible-infra-automation.git
cd ansible-infra-automation

# Verify directory structure
ls -la
```

## Phase 3: Ansible Execution

### 3.1 Test Ansible Connectivity

```bash
cd /opt/ansible-infra-automation

# Test ping to all hosts
ansible all -m ping

# Expected output:
# web01 | SUCCESS => { "ping": "pong" }
# web02 | SUCCESS => { "ping": "pong" }
# storage01 | SUCCESS => { "ping": "pong" }
# db-central01 | SUCCESS => { "ping": "pong" }
```

### 3.2 Run Complete Infrastructure Setup

**Option A: Full Deployment (Recommended)**

```bash
# Deploy entire infrastructure in correct order
ansible-playbook all_setup.yml

# Execution time: ~15-20 minutes
```

**Option B: Component-by-Component**

```bash
# 1. Storage first (required by web servers)
ansible-playbook storage_setup.yml

# 2. Database second (required by WordPress)
ansible-playbook database_setup.yml

# 3. Web servers (depend on storage and database)
ansible-playbook web_setup.yml

# 4. Load balancer last (requires backends)
ansible-playbook lb_setup.yml
```

### 3.3 Monitor Execution

During playbook execution, watch for:
- **CHANGED**: Task modified system state (expected)
- **OK**: Task verified state (idempotent, safe)
- **FAILED**: Task encountered error (requires investigation)
- **SKIPPED**: Task not applicable

Successful run shows:
```
PLAY RECAP *********************************************************************
lb01                       : ok=XX   changed=XX   unreachable=0    failed=0
web01                      : ok=XX   changed=XX   unreachable=0    failed=0
web02                      : ok=XX   changed=XX   unreachable=0    failed=0
storage01                  : ok=XX   changed=XX   unreachable=0    failed=0
db-central01               : ok=XX   changed=XX   unreachable=0    failed=0
```

## Phase 4: Verification

### 4.1 Check Load Balancer

```bash
# Test HAProxy
curl http://192.168.56.11

# View HAProxy stats
# Open browser: http://192.168.56.11:8404/stats
# Username: admin
# Password: admin123

# Test DNS
dig @192.168.56.11 example.com
nslookup www.example.com 192.168.56.11
```

### 4.2 Check Web Servers

```bash
# Test WEB01
curl http://192.168.56.21        # Port 80
curl http://192.168.56.21:8080   # Port 8080

# Test WEB02
curl http://192.168.56.22        # Port 80
curl http://192.168.56.22:8080   # Port 8080

# Check NFS mount
ssh root@192.168.56.21 'df -h | grep nfs'
ssh root@192.168.56.22 'df -h | grep nfs'
```

### 4.3 Check Storage Server

```bash
# Verify NFS exports
ssh root@192.168.56.30 'exportfs -v'

# Expected output:
# /srv/nfs/shared 192.168.56.0/24(rw,sync,no_root_squash,no_subtree_check)

# Check NFS service
ssh root@192.168.56.30 'systemctl status nfs-server'
```

### 4.4 Check Database Server

```bash
# Test database connection
ssh root@192.168.56.40 'mysql -e "SHOW DATABASES;"'

# Verify WordPress database
ssh root@192.168.56.40 'mysql -e "USE wordpress; SHOW TABLES;"'

# Test remote connection from WEB01
ssh root@192.168.56.21 "mysql -h 192.168.56.40 -u wpuser -p'SecurePassword123!' -e 'SELECT 1;'"
```

### 4.5 Check WordPress

```bash
# Test WordPress URL
curl http://192.168.56.21/wordpress/

# Complete installation via browser:
# http://192.168.56.21/wordpress/wp-admin/install.php

# Or through load balancer:
# http://192.168.56.11/wordpress/
```

## Phase 5: WordPress Installation (Web UI)

1. Open browser: `http://192.168.56.11/wordpress/`
2. Select language: English
3. Click "Let's go!"
4. **Database Configuration** (pre-filled via Ansible):
   - Database Name: `wordpress`
   - Username: `wpuser`
   - Password: `SecurePassword123!`
   - Database Host: `192.168.56.40`
   - Table Prefix: `wp_`
5. Click "Submit"
6. Click "Run the installation"
7. **Site Information**:
   - Site Title: Your site name
   - Username: admin
   - Password: (strong password)
   - Email: your@email.com
8. Click "Install WordPress"
9. Log in and verify dashboard access

## Phase 6: Testing Load Balancing

### 6.1 Manual Failover Test

```bash
# Stop WEB01 Apache
ssh root@192.168.56.21 'systemctl stop httpd'

# Test load balancer (should still work, served by WEB02)
curl http://192.168.56.11

# Check HAProxy stats - WEB01 should show as DOWN
# Browser: http://192.168.56.11:8404/stats

# Restart WEB01
ssh root@192.168.56.21 'systemctl start httpd'

# Wait 10 seconds, verify WEB01 is UP in HAProxy stats
```

### 6.2 Load Distribution Test

```bash
# Run multiple requests
for i in {1..20}; do
  curl -s http://192.168.56.11 | grep -oP '(?<=Server: )[^<]+'
done

# Should see both web01 and web02 in output (round-robin)
```

## Phase 7: Ongoing Maintenance

### 7.1 Re-run Playbooks (Idempotent)

```bash
# Safe to run multiple times - Ansible ensures idempotency
ansible-playbook all_setup.yml

# Run with tags (specific components)
ansible-playbook all_setup.yml --tags "web"
ansible-playbook all_setup.yml --tags "database"
```

### 7.2 Update Configuration

1. Edit variable files:
   - `group_vars/all.yml`
   - `host_vars/web01.yml`
   - Template files in `roles/*/templates/`

2. Re-run playbook:
```bash
ansible-playbook all_setup.yml
```

### 7.3 Check Logs

```bash
# Apache logs
ssh root@192.168.56.21 'tail -f /var/log/httpd/error_log'

# MariaDB logs
ssh root@192.168.56.40 'tail -f /var/log/mariadb/mariadb.log'

# HAProxy logs
ssh root@192.168.56.11 'journalctl -u haproxy -f'

# NFS logs
ssh root@192.168.56.30 'journalctl -u nfs-server -f'
```

## Common Issues & Solutions

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for detailed troubleshooting steps.

## Next Steps

- Configure SSL/TLS certificates
- Set up monitoring with Prometheus + Grafana
- Implement centralized logging (ELK Stack)
- Add backup scripts for database and NFS
- Configure log rotation
- Implement CI/CD pipeline for automated deployments
