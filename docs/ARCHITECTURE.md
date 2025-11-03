# System Architecture

## Overview

This project implements a complete multi-tier infrastructure using Ansible automation on Rocky Linux 9.

## Architecture Diagram

```
                    Internet
                       |
                       |
        +--------------v--------------+
        |                             |
        |   Load Balancer + DNS Tier  |
        |                             |
        |   LB01 (192.168.56.11)     |
        |   - HAProxy (Port 80)       |
        |   - BIND DNS (Port 53)      |
        |   - Stats (Port 8404)       |
        |                             |
        +-------------+---------------+
                      |
          +-----------+-----------+
          |                       |
+---------v---------+   +---------v---------+
|                   |   |                   |
|  Application Tier |   |  Application Tier |
|                   |   |                   |
|  WEB01            |   |  WEB02            |
| (.56.21)          |   | (.56.22)          |
|  - Apache 80,8080 |   |  - Apache 80,8080 |
|  - PHP + WP       |   |  - Static Web     |
|  - NFS Client     |   |  - NFS Client     |
|                   |   |                   |
+---------+---------+   +---------+---------+
          |                       |
          +-------+-------+-------+
                  |       |
          +-------v--+  +-v-------+
          |          |  |         |
          | Storage  |  | Database|
          | Tier     |  | Tier    |
          |          |  |         |
          | Storage01|  |DB-central01
          |(.56.30)  |  |(.56.40) |
          |- NFS     |  |- MariaDB|
          |  Server  |  |  MySQL  |
          |          |  |         |
          +----------+  +---------+
```

## Component Details

### 1. Load Balancer Tier (LB01)

**IP Address**: 192.168.56.11

**Services**:
- **HAProxy**: Layer 7 load balancer
  - Round-robin algorithm
  - Health checks every 5 seconds
  - Automatic backend failover
  - Statistics dashboard on port 8404

- **BIND DNS**: Authoritative DNS server
  - Zone: example.com
  - A records for all infrastructure components
  - SOA and NS records

**Key Features**:
- Single point of entry for all web traffic
- Automatic detection of unhealthy backends
- 30-second recovery time on backend failure
- Real-time traffic distribution monitoring

### 2. Application Tier (WEB01, WEB02)

#### WEB01 (192.168.56.21)
**Primary Services**:
- Apache HTTP Server (ports 80, 8080)
- PHP 7.x with extensions
- WordPress application
- NFS client for shared storage

**VirtualHosts**:
1. Port 80: WordPress site (`wordpress.example.com`)
2. Port 8080: Static site 1 (`web01.example.com`)

#### WEB02 (192.168.56.22)
**Primary Services**:
- Apache HTTP Server (ports 80, 8080)
- Static web content
- NFS client for shared storage

**VirtualHosts**:
1. Port 80: Main site (`www.example.com`)
2. Port 8080: Static site 2 (`web02.example.com`)

**Shared Features**:
- SELinux enforcing mode with custom policies
- NFS-mounted shared storage
- Firewalld with HTTP/HTTPS rules
- Apache MPM prefork for PHP compatibility

### 3. Storage Tier (Storage01)

**IP Address**: 192.168.56.30

**Services**:
- NFS Server (NFSv3)
- Export path: `/srv/nfs/shared`
- Exported to: 192.168.56.0/24 network

**Features**:
- Read-write access for web servers
- No root squash for admin operations
- Firewalld rules for NFS, RPC-bind, mountd
- Persistent mounts on web servers via `/etc/fstab`

**Use Cases**:
- Shared WordPress uploads
- Configuration file synchronization
- Centralized static assets

### 4. Database Tier (DB-central01)

**IP Address**: 192.168.56.40

**Services**:
- MariaDB 10.x
- WordPress database
- Remote access enabled

**Configuration**:
- Bind address: 0.0.0.0 (all interfaces)
- Character set: UTF-8 (utf8mb4)
- InnoDB engine with optimized settings
- Binary logging enabled
- Slow query logging

**Security**:
- User-specific credentials
- Host-based access control (192.168.56.%)
- Firewalld rule for port 3306
- Root password protection

**Database Details**:
- Database name: wordpress
- User: wpuser
- Collation: utf8mb4_unicode_ci

## Network Configuration

### Network Segments

1. **NAT Network**: `10.0.2.0/24`
   - Purpose: Internet access
   - Used by: All VMs

2. **Host-Only Network**: `192.168.56.0/24`
   - Purpose: Internal communication
   - Used by: All infrastructure components

### Firewall Rules

| Component | Ports | Protocol | Purpose |
|-----------|-------|----------|---------|
| LB01 | 80 | TCP | HTTP |
| LB01 | 8404 | TCP | HAProxy Stats |
| LB01 | 53 | TCP/UDP | DNS |
| WEB01/02 | 80, 8080 | TCP | HTTP |
| Storage01 | 2049 | TCP/UDP | NFS |
| Storage01 | 111 | TCP/UDP | RPC-bind |
| Storage01 | 20048 | TCP/UDP | Mountd |
| DB-central01 | 3306 | TCP | MySQL |

## High Availability Features

### Load Balancer
- Health checks every 5 seconds
- Automatic backend removal on 3 consecutive failures
- Backend re-addition after 2 consecutive successes
- Connection retry with 10s timeout

### Web Servers
- Multi-instance architecture (2 servers)
- Shared storage via NFS
- Session persistence possible via shared filesystem
- Independent failure domains

### Data Persistence
- NFS for shared files
- MariaDB with binary logging
- Backup capability via NFS snapshots

## Security Measures

### SELinux
- Enforcing mode on all servers
- Custom booleans:
  - `httpd_can_network_connect`: Apache external connections
  - `httpd_can_network_connect_db`: Apache to database
  - `httpd_use_nfs`: Apache NFS access
  - `haproxy_connect_any`: HAProxy backend connections

### Firewalld
- Default deny policy
- Whitelist-based rules
- Service-specific zones
- Immediate rule activation

### Application Security
- WordPress salts from official API
- Database credentials in protected config files
- File permissions (640 for configs, 755 for directories)
- Disabled file editing in WordPress admin

## Performance Characteristics

### Expected Performance
- **Concurrent Users**: 500+ (limited by single LB)
- **Response Time**: <100ms (static), <500ms (WordPress)
- **Failover Time**: 30 seconds
- **Recovery Time**: 10 seconds

### Bottlenecks
1. Single load balancer (no HA pair)
2. NFS as single point of failure
3. Single database server (no replication)

### Scalability
- Horizontal: Add more web servers to HAProxy backend
- Vertical: Increase VM resources (CPU, RAM)
- Database: Add read replicas (requires code changes)

## Monitoring Points

### HAProxy Statistics
- URL: http://192.168.56.11:8404/stats
- Metrics: Backend status, queue depth, session count
- Refresh: 30 seconds

### System Monitoring (Manual)
- Apache logs: `/var/log/httpd/`
- MariaDB logs: `/var/log/mariadb/`
- NFS logs: `journalctl -u nfs-server`
- HAProxy logs: `journalctl -u haproxy`

## AWS Equivalent Architecture

| On-Premise Component | AWS Service |
|---------------------|-------------|
| HAProxy | Application Load Balancer (ALB) |
| BIND DNS | Route 53 |
| Apache on EC2 | EC2 Auto Scaling Group |
| NFS | Elastic File System (EFS) |
| MariaDB | RDS Multi-AZ (MariaDB/MySQL) |
| VirtualBox Network | VPC with Subnets |
| Firewalld | Security Groups |

## Deployment Order

1. Storage (NFS must be available first)
2. Database (required by WordPress)
3. Web Servers (depend on NFS and DB)
4. Load Balancer (must have backends ready)

This order ensures dependencies are met and services start cleanly.
