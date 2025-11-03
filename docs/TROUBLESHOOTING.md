# Troubleshooting Guide

This document covers common issues and their solutions.

## Issue 1: Apache Cannot Listen on Port 8080

### Symptoms
- Apache starts successfully on port 80
- Port 8080 not accessible: `curl: (7) Failed to connect to 192.168.56.21 port 8080`
- Error in `/var/log/httpd/error_log`: `Address already in use: AH00072: make_sock: could not bind to address [::]:8080`

### Root Cause
- Port 8080 already in use by another process
- Firewall blocking port 8080
- SELinux preventing Apache from binding to port 8080

### Solution

#### Step 1: Check Port Usage
```bash
sudo ss -tlnp | grep 8080
# OR
sudo lsof -i :8080
```

If another process is using port 8080:
```bash
# Stop the conflicting service
sudo systemctl stop <service_name>

# OR kill the process
sudo kill -9 <PID>
```

#### Step 2: Configure Apache
Ensure `httpd.conf` has Listen directive:
```bash
sudo grep -n "Listen" /etc/httpd/conf/httpd.conf
```

Should show:
```
Listen 80
Listen 8080
```

If missing, add:
```bash
sudo sed -i '/^Listen 80/a Listen 8080' /etc/httpd/conf/httpd.conf
```

#### Step 3: Configure Firewall
```bash
# Add rule
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

#### Step 4: Configure SELinux (if enforcing)
```bash
# Allow Apache to listen on port 8080
sudo semanage port -a -t http_port_t -p tcp 8080

# Verify
sudo semanage port -l | grep http_port_t
```

#### Step 5: Restart Apache
```bash
sudo systemctl restart httpd
sudo systemctl status httpd
```

#### Step 6: Verify
```bash
curl http://192.168.56.21:8080
```

### Ansible Fix
The role already handles this. Re-run:
```bash
ansible-playbook web_setup.yml
```

---

## Issue 2: NFS Mount Failure on Web Servers

### Symptoms
- Web server cannot access `/srv/web/shared`
- Error: `mount.nfs: access denied by server while mounting`
- Apache shows permission denied errors
- SELinux AVC denials in audit log

### Root Cause
- NFS server not exporting directory
- Network connectivity issues
- Firewall blocking NFS ports
- SELinux blocking Apache NFS access
- Incorrect NFS export permissions

### Solution

#### Step 1: Verify NFS Server Status
```bash
# On Storage01
ssh root@192.168.56.30 'systemctl status nfs-server'
ssh root@192.168.56.30 'exportfs -v'
```

Expected output:
```
/srv/nfs/shared 192.168.56.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,
  no_root_squash,no_all_squash,no_subtree_check,secure_locks,acl,
  no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,no_root_squash,
  no_all_squash)
```

#### Step 2: Test Network Connectivity
```bash
# From web server
ping -c 3 192.168.56.30
showmount -e 192.168.56.30
```

If showmount fails:
```bash
# On Storage01 - Open firewall
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

#### Step 3: Manual Mount Test
```bash
# On web server
sudo mkdir -p /srv/web/shared
sudo mount -t nfs 192.168.56.30:/srv/nfs/shared /srv/web/shared

# Check mount
df -h | grep nfs
```

If mount succeeds but files not accessible:
```bash
# Test write permission
sudo touch /srv/web/shared/test.txt
ls -la /srv/web/shared/
```

#### Step 4: Configure SELinux on Web Servers
```bash
# Enable NFS for Apache
sudo setsebool -P httpd_use_nfs 1

# Verify
getsebool httpd_use_nfs
# Output: httpd_use_nfs --> on
```

#### Step 5: Make Mount Persistent
```bash
# Add to /etc/fstab
echo "192.168.56.30:/srv/nfs/shared /srv/web/shared nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Test fstab
sudo umount /srv/web/shared
sudo mount -a
df -h | grep nfs
```

#### Step 6: Set Correct Ownership
```bash
# On web servers
sudo chown apache:apache /srv/web/shared
sudo chmod 755 /srv/web/shared
```

### Ansible Fix
Re-run storage and web playbooks:
```bash
ansible-playbook storage_setup.yml
ansible-playbook web_setup.yml
```

---

## Issue 3: MariaDB Remote Connection Refused

### Symptoms
- WordPress cannot connect to database
- Error: `ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.56.40' (111)`
- MariaDB only listening on localhost
- Connection timeout from web servers

### Root Cause
- MariaDB bound to 127.0.0.1 (localhost only)
- Firewall blocking port 3306
- User host permissions incorrect
- MariaDB not started

### Solution

#### Step 1: Check MariaDB Status
```bash
# On DB-central01
ssh root@192.168.56.40 'systemctl status mariadb'
```

If not running:
```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

#### Step 2: Verify Bind Address
```bash
# Check current bind address
ssh root@192.168.56.40 "grep bind-address /etc/my.cnf.d/*.cnf"
```

Should be:
```
bind-address = 0.0.0.0
```

If shows `127.0.0.1` or missing:
```bash
# Edit configuration
sudo vi /etc/my.cnf.d/server.cnf

# Add under [mysqld]:
[mysqld]
bind-address = 0.0.0.0
```

Restart MariaDB:
```bash
sudo systemctl restart mariadb
```

#### Step 3: Verify Listening Port
```bash
ssh root@192.168.56.40 'ss -tlnp | grep 3306'
```

Should show:
```
LISTEN 0      80           0.0.0.0:3306       0.0.0.0:*    users:(("mariadbd",pid=XXXX,fd=XX))
```

If shows `127.0.0.1:3306`, bind-address is still localhost.

#### Step 4: Configure Firewall
```bash
# On DB-central01
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

#### Step 5: Create User with Correct Permissions
```bash
# On DB-central01
sudo mysql -e "CREATE USER IF NOT EXISTS 'wpuser'@'192.168.56.%' IDENTIFIED BY 'SecurePassword123!';"
sudo mysql -e "GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'192.168.56.%';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Verify
sudo mysql -e "SELECT user, host FROM mysql.user WHERE user='wpuser';"
```

Expected output:
```
+--------+--------------+
| user   | host         |
+--------+--------------+
| wpuser | 192.168.56.% |
+--------+--------------+
```

#### Step 6: Test Connection from Web Server
```bash
# From WEB01
mysql -h 192.168.56.40 -u wpuser -p'SecurePassword123!' -e "SELECT 1;"
```

Expected output:
```
+---+
| 1 |
+---+
| 1 |
+---+
```

### Ansible Fix
```bash
ansible-playbook database_setup.yml
```

---

## Issue 4: HAProxy Backend Shows DOWN

### Symptoms
- HAProxy stats page shows backend as DOWN
- Red indicator in HAProxy dashboard
- Health check failures
- Requests not distributed to affected backend

### Root Cause
- Apache not running on backend server
- Firewall blocking port 80 on backend
- Backend server network issue
- Health check URL returns non-200 status

### Solution

#### Step 1: Check Backend Apache Status
```bash
# On web server
ssh root@192.168.56.21 'systemctl status httpd'
```

If not running:
```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

#### Step 2: Test Backend Directly
```bash
# From LB01 or your machine
curl -I http://192.168.56.21/
```

Should return:
```
HTTP/1.1 200 OK
```

If connection refused or timeout:
```bash
# On web server - check firewall
sudo firewall-cmd --list-services
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

#### Step 3: Verify HAProxy Configuration
```bash
# On LB01
sudo cat /etc/haproxy/haproxy.cfg | grep -A 5 "backend http_back"
```

Should show:
```
backend http_back
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server web01 192.168.56.21:80 check inter 5s rise 2 fall 3
    server web02 192.168.56.22:80 check inter 5s rise 2 fall 3
```

#### Step 4: Test Health Check Manually
```bash
# From LB01
curl -I http://192.168.56.21/
curl -I http://192.168.56.22/
```

Both should return `HTTP/1.1 200 OK`.

#### Step 5: Check HAProxy Logs
```bash
# On LB01
sudo journalctl -u haproxy -n 50 --no-pager | grep -i "health\|check"
```

Look for messages like:
- `Health check for server http_back/web01 succeeded`
- `Server http_back/web01 is UP`

#### Step 6: Force Backend State Change
```bash
# Restart HAProxy
sudo systemctl restart haproxy

# Wait 10 seconds for health checks
sleep 10

# Check stats
curl http://admin:admin123@192.168.56.11:8404/stats
```

### Ansible Fix
```bash
# Re-run web and load balancer playbooks
ansible-playbook web_setup.yml
ansible-playbook lb_setup.yml
```

---

## Issue 5: DNS Resolution Not Working

### Symptoms
- `dig @192.168.56.11 example.com` returns `REFUSED` or timeout
- Cannot resolve internal hostnames
- BIND service not running

### Root Cause
- BIND not started
- Firewall blocking DNS port (53)
- Zone file syntax errors
- Named configuration errors

### Solution

#### Step 1: Check BIND Status
```bash
# On LB01
sudo systemctl status named
```

If failed to start, check logs:
```bash
sudo journalctl -u named -n 50 --no-pager
```

#### Step 2: Validate Configuration Files
```bash
# Check named.conf syntax
sudo named-checkconf /etc/named.conf

# Check zone file syntax
sudo named-checkzone example.com /var/named/zones/db.example.com
```

If errors found, fix syntax and restart:
```bash
sudo systemctl restart named
```

#### Step 3: Open Firewall
```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

#### Step 4: Test DNS Queries
```bash
# Query from another machine
dig @192.168.56.11 example.com
dig @192.168.56.11 www.example.com
dig @192.168.56.11 wordpress.example.com

# Should return A records with correct IPs
```

#### Step 5: Configure Clients to Use DNS
```bash
# On web servers, edit /etc/resolv.conf
echo "nameserver 192.168.56.11" | sudo tee /etc/resolv.conf

# Test
nslookup www.example.com
```

### Ansible Fix
```bash
ansible-playbook lb_setup.yml --tags "dns"
```

---

## General Debugging Commands

### Check All Services Status
```bash
# Run on LB01 (Ansible control node)
ansible all -m shell -a "systemctl status httpd" -i inventory/hosts.ini
ansible database -m shell -a "systemctl status mariadb"
ansible storage -m shell -a "systemctl status nfs-server"
ansible lb -m shell -a "systemctl status haproxy"
ansible lb -m shell -a "systemctl status named"
```

### Check Firewall Rules
```bash
ansible all -m shell -a "firewall-cmd --list-all"
```

### Check SELinux Denials
```bash
ansible all -m shell -a "ausearch -m AVC -ts recent"
```

### Verify Network Connectivity
```bash
ansible all -m ping
ansible all -m shell -a "ip addr show"
```

### Check Disk Space
```bash
ansible all -m shell -a "df -h"
```

### Check Logs
```bash
# Apache
ansible webservers -m shell -a "tail -20 /var/log/httpd/error_log"

# MariaDB
ansible database -m shell -a "tail -20 /var/log/mariadb/mariadb.log"

# System logs
ansible all -m shell -a "journalctl -p err -n 20 --no-pager"
```

---

## Getting Help

If issues persist:

1. **Check Ansible verbose output**:
   ```bash
   ansible-playbook all_setup.yml -v   # Verbose
   ansible-playbook all_setup.yml -vv  # More verbose
   ansible-playbook all_setup.yml -vvv # Debug level
   ```

2. **Run syntax check**:
   ```bash
   ansible-playbook all_setup.yml --syntax-check
   ```

3. **Dry run (check mode)**:
   ```bash
   ansible-playbook all_setup.yml --check
   ```

4. **Review role documentation**:
   - Ansible documentation: https://docs.ansible.com/
   - HAProxy: http://www.haproxy.org/docs.html
   - Apache: https://httpd.apache.org/docs/
   - MariaDB: https://mariadb.com/kb/en/documentation/

5. **Contact**:
   - GitHub Issues: https://github.com/qkrtpdlr/ansible-infra-automation/issues
   - Email: rlagudfo1223@gmail.com
