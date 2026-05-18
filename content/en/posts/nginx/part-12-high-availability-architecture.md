---
title: "Nginx from Beginner to Pro · Part 12: High Availability Architecture"
date: 2025-01-03
weight: 1112
draft: false
tags: ["nginx"]
---

## The Single Point of Failure Problem

Nginx itself can go down. With only one Nginx instance, the entire service becomes unavailable if it fails.

**Solution**: Nginx active-standby mode — two Nginx instances, one serving traffic while the other monitors. When the primary fails, the standby automatically takes over.

```
                    ┌─ Nginx Primary (192.168.1.10)
User → VIP (10.0.0.100) ─┼─ Nginx Standby (192.168.1.11)
                    └─ Actual request path
```

Users access a Virtual IP (VIP), which always points to the currently available Nginx.

---

## Keepalived Introduction

Keepalived is a high-availability solution implemented through **VRRP (Virtual Router Redundancy Protocol)**:

- The primary node sends periodic heartbeats (VRRP broadcasts)
- If the standby receives no heartbeat, it takes over the VIP
- VIP switching is transparent to clients

---

## Installing Keepalived

```bash
# Ubuntu/Debian
sudo apt install keepalived -y

# CentOS/RHEL
sudo yum install keepalived -y
```

---

## Primary Node Configuration

```bash
# /etc/keepalived/keepalived.conf

global_defs {
    router_id NGINX_MASTER      # Route identifier, must be unique
    script_user root
    enable_script_security
}

# Health check script
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2                  # Check every 2 seconds
    timeout 3                   # Script timeout 3 seconds
    weight -50                  # Weight decrease on failure
    rise 2                      # 2 consecutive successes = healthy
    fall 3                      # 3 consecutive failures = unhealthy
}

vrrp_instance VI_1 {
    state MASTER                # Primary node
    interface eth0              # Network interface
    virtual_router_id 51        # Virtual router ID (must match standby)
    priority 150                # Priority, higher is more likely to be master
    advert_int 1                # Heartbeat interval (seconds)
    nopreempt                   # Don't preempt (primary won't reclaim VIP on recovery)

    authentication {
        auth_type PASS
        auth_pass 1234          # Authentication password
    }

    virtual_ipaddress {
        10.0.0.100/24 dev eth0 label eth0:vip  # VIP
    }

    track_script {
        check_nginx
    }

    notify_master /etc/keepalived/scripts/master.sh   # Run on becoming master
    notify_backup /etc/keepalived/scripts/backup.sh   # Run on becoming backup
    notify_fault  /etc/keepalived/scripts/fault.sh    # Run on fault
}
```

---

## Standby Node Configuration

```bash
# /etc/keepalived/keepalived.conf (standby node)

global_defs {
    router_id NGINX_BACKUP      # Different from primary
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    timeout 3
    weight -50
}

vrrp_instance VI_1 {
    state BACKUP                # Standby node
    interface eth0
    virtual_router_id 51        # Must match primary
    priority 100                # Lower priority than primary
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        10.0.0.100/24 dev eth0 label eth0:vip
    }

    track_script {
        check_nginx
    }
}
```

---

## Nginx Health Check Script

```bash
#!/bin/bash
# /etc/keepalived/check_nginx.sh

# Method 1: Check if process exists
if ! pgrep -x nginx > /dev/null; then
    # Try to restart Nginx
    systemctl start nginx

    # Wait 2 seconds then recheck
    sleep 2

    # If restart fails, let Keepalived trigger failover
    if ! pgrep -x nginx > /dev/null; then
        exit 1
    fi
fi

# Method 2: Check HTTP response (more reliable)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/health 2>/dev/null)
if [ "$HTTP_CODE" != "200" ]; then
    systemctl restart nginx
    sleep 2
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/health 2>/dev/null)
    if [ "$HTTP_CODE" != "200" ]; then
        exit 1
    fi
fi

exit 0
```

```bash
# Make executable
chmod +x /etc/keepalived/check_nginx.sh
```

---

## Starting & Verifying

```bash
# Start Keepalived (on both primary and standby)
sudo systemctl start keepalived
sudo systemctl enable keepalived

# Check which machine holds the VIP
ip addr show eth0 | grep 10.0.0.100

# Check Keepalived status
sudo systemctl status keepalived

# View VRRP logs
journalctl -u keepalived -f

# Check VIP binding
ip addr show dev eth0
```

Log output shows the primary-standby interaction:

```
Feb 15 10:00:01 master Keepalived_vrrp[1234]: Sending gratuitous ARP on eth0 for 10.0.0.100
Feb 15 10:00:01 master Keepalived_vrrp[1234]: VRRP_Instance(VI_1) Transition to MASTER STATE
```

---

## Testing Failover

```bash
# Stop Nginx on the primary node
sudo systemctl stop nginx

# Observe logs
journalctl -u keepalived -f

# Check if VIP has moved to the standby node
# Run on standby node
ip addr show eth0 | grep 10.0.0.100

# Restore Nginx on the primary node
sudo systemctl start nginx
```

With `nopreempt` enabled, the recovered primary won't reclaim the VIP, avoiding unnecessary switching.

---

## Active-Active Mode (Load Balancing + HA)

Both Nginx nodes serve traffic while backing each other up:

```
               VIP1: 10.0.0.100 (primary: Node1, standby: Node2)
User Request ────┤
               VIP2: 10.0.0.101 (primary: Node2, standby: Node1)
```

### Node1 Configuration

```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }
}

vrrp_instance VI_2 {
    state BACKUP
    interface eth0
    virtual_router_id 52
    priority 100
    virtual_ipaddress {
        10.0.0.101/24 dev eth0
    }
}
```

### Node2 Configuration

```bash
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }
}

vrrp_instance VI_2 {
    state MASTER
    interface eth0
    virtual_router_id 52
    priority 150
    virtual_ipaddress {
        10.0.0.101/24 dev eth0
    }
}
```

Combine with DNS round-robin or SLB — traffic is distributed evenly across both Nginx nodes, and if either fails, the other takes over all traffic.

---

## Nginx Configuration Sync

Both primary and standby Nginx need consistent configuration. Recommended approaches:

### Approach 1: Version Control + Manual Sync

```bash
# Manage config with git
cd /etc/nginx
git init
git add .
git commit -m "init nginx config"

# Pull on standby server
git pull origin main
nginx -t && systemctl reload nginx
```

### Approach 2: lsyncd Real-Time Sync

```bash
# Install lsyncd
sudo apt install lsyncd

# /etc/lsyncd/lsyncd.conf
settings {
    logfile = "/var/log/lsyncd/lsyncd.log",
    statusFile = "/var/log/lsyncd/lsyncd.status"
}

sync {
    default.rsync,
    source = "/etc/nginx/",
    target = "10.0.0.11:/etc/nginx/",
    exclude = { ".git" }
}
```

### Approach 3: Configuration Management Tools

Use Ansible, SaltStack, or similar tools for unified configuration management.

---

## DNS Round-Robin + Keepalived

Configure multiple DNS A records for the VIP, combined with Keepalived:

```
example.com A 10.0.0.100
example.com A 10.0.0.101
example.com A 10.0.0.102
```

Each IP is backed by a Keepalived active-standby pair.

---

## Complete HA Architecture

```
                      ┌─ DNS / SLB
                      │
              ┌───────┴────────┐
              │                │
         Nginx Primary     Nginx Standby
        (10.0.0.10)      (10.0.0.11)
              │                │
              └───────┬────────┘
                      │ VIP: 10.0.0.100
                      │
              ┌───────┴────────┐
              │                │
         App Server 1     App Server 2
         App Server 3     App Server 4
              │                │
              └───────┬────────┘
                      │
                   Database
                 (Primary/Replica)
```

**Failure Scenarios**:

| Failure | Impact | Recovery |
|---------|--------|----------|
| Primary Nginx down | VIP fails over to standby (seconds) | Automatic |
| Primary Nginx recovered | Standby continues serving (nopreempt) | Manual or automatic |
| Backend server down | Load balancer removes it automatically | Automatic |
| Entire machine down | VIP failover + backend failover | Automatic |

---

## Summary

This article covered using Keepalived to implement Nginx active-standby high availability, including configuration, health checks, failover testing, and configuration synchronization.

---

