---
title: "SSH Brute Force Incident Response — When 500,000 Login Attempts Hit Your Bastion"
date: 2026-05-30
weight: 100230
slug: "ssh-brute-force-incident-response"
tags: ["security", "ssh", "linux", "incident-response", "troubleshooting"]
categories: ["Security"]
description: "An SSH brute force attack incident — how a publicly exposed bastion host with default settings survived 500,000 login attempts, and why fail2ban + SSH key auth + Cloudflare are the minimum viable defense"
keywords: "ssh brute force attack, fail2ban config, linux ssh security, bastion host protection, kubernetes node ssh access"
draft: false
featured: true
cover:
  image: ""
  caption: "SSH Brute Force — Incident Response"
---

# SSH Brute Force Incident Response — When 500,000 Login Attempts Hit Your Bastion

## Common Search Queries

- ssh brute force attack stopped
- fail2ban ssh configuration
- linux server under ssh attack
- bastion host security best practices
- kubernetes node ssh access security

---

## The Incident

**Environment**: Single bastion host (4C/8G, Ubuntu 22.04), used as SSH jump box for 5 Kubernetes cluster nodes. Publicly accessible on port 22.

**Time**: Sunday 03:00 AM. Cloudflare Security Center alerted: anomalous traffic spike to the bastion host's SSH port.

**Initial Symptoms**: The bastion host's CPU was pegged at 100%. SSH connections from legitimate admins were timing out. The system was nearly unresponsive.

```bash
# What the admin saw when trying to connect
ssh admin@bastion.example.com
# Connection timed out after 30 seconds
# Tried again — same result
```

**Impact**: 45-minute loss of access to all 5 Kubernetes clusters. Emergency out-of-band access (iDRAC/IPMI) had to be used for incident response.

---

## Background

The bastion host had been running for 18 months without issue. It was configured with:

- Password authentication enabled (for "emergency fallback")
- Default SSH port (22)
- No rate limiting
- No fail2ban
- Standard Ubuntu install with default sshd_config

Over 18 months, the attack volume had gradually increased. What started as a few hundred attempts per day had grown to tens of thousands. The host handled it — until this Sunday.

An attacker had found the host through Shodan scanning and added it to a distributed brute force botnet. The attack peaked at 500,000 login attempts in 2 hours from 1,200+ unique IPs.

```bash
# Check current failed login count
grep "Failed password" /var/log/auth.log | wc -l
# 512,347 failures in the last 2 hours
```

---

## Investigation

### Step 1: Assess the Damage

```bash
# Check for successful logins
grep "Accepted" /var/log/auth.log | grep -v "CRON" | grep -v "systemd"
# Result: 0 successful non-admin logins
```

Fortunately, no attacker had successfully logged in. The damage was purely availability: the CPU was overwhelmed processing authentication requests.

### Step 2: Identify the Attack Pattern

```bash
# Top attacking IPs
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -10
```

```
48291 45.33.32.156
47321 185.220.101.42
44107 51.75.145.203
...
```

1,200+ unique IPs. Coordinated distributed brute force.

### Step 3: Check for Credential Stuffing

```bash
# Most commonly tried usernames
grep "Failed password" /var/log/auth.log | awk '{print $(NF-5)}' | sort | uniq -c | sort -rn | head -10
```

```
112034 root
98431 admin
83721 ubuntu
...
```

Standard credential stuffing: `root`, `admin`, `ubuntu`, `test`, `deploy`, `jenkins`.

---

## Root Cause

1. **Password authentication enabled**: SSH key authentication was the primary method, but password auth was left on as "backup"
2. **No rate limiting**: Every failed attempt consumed CPU cycles for authentication processing
3. **No fail2ban or similar tool**: 500,000 attempts before anyone noticed
4. **Public SSH access without DDoS protection**: Bastion directly exposed to the internet with no WAF or DDoS mitigation layer
5. **No monitoring on auth.log anomalies**: No alerting for unusual spikes in failed SSH attempts

---

## Resolution

### Emergency (Immediate)

```bash
# 1. Kill all current SSH sessions to free CPU
pkill -9 sshd
systemctl restart sshd

# 2. Temporarily block all traffic except from known office IPs
iptables -A INPUT -p tcp --dport 22 -s <OFFICE_IP_1> -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -s <OFFICE_IP_2> -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

Within 1 minute, CPU dropped from 100% to 5%.

### Install and Configure fail2ban

```bash
apt install fail2ban -y

cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 300
ignoreip = <OFFICE_IP_1> <OFFICE_IP_2>
EOF

systemctl enable --now fail2ban
```

### Harden SSH Configuration

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
PermitRootLogin no
MaxAuthTries 3
MaxSessions 10
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
```

### Move SSH to Alternative Port (Defense in Depth)

```bash
# /etc/ssh/sshd_config
Port 22222

systemctl restart sshd
```

### Add Cloudflare Spectrum or Similar DDoS Protection

Configured Cloudflare Spectrum to proxy SSH traffic, hiding the real bastion IP:

```
Cloudflare → Spectrum (SSH proxy) → Bastion Host (port 22222)
```

### Deploy CrowdSec for Advanced Threat Intelligence

```bash
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | bash
apt install crowdsec crowdsec-firewall-bouncer-iptables -y
```

CrowdSec shares attack data with a global threat intelligence network — an IP attacking your bastion gets blocked everywhere, not just on your host.

### Kubernetes Cluster Access Audit

```bash
# Check all kubeconfig files on the bastion
find /home -name "kubeconfig" -o -name "*.kubeconfig" 2>/dev/null

# Rotate cluster admin credentials
kubectl config view --flatten > /tmp/admin-kubeconfig
# Revoke and re-issue all kubeconfigs
```

### Monitoring

```bash
# Alert on SSH failure spikes
# Prometheus + node_exporter + alertmanager rule:
groups:
- name: ssh-security
  rules:
  - alert: SSHFailureSpike
    expr: rate(node_ssh_failures_total[5m]) > 10
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "SSH failure rate spike detected (>10/s)"
```

---

## Lessons Learned

- **Password auth is a liability, not a backup**: SSH keys are more secure and consume negligible CPU. Password auth opens CPU exhaustion attack surface
- **fail2ban is table stakes**: Every publicly accessible SSH server needs fail2ban or equivalent. Default config with 3 retries and 1-hour ban is the minimum
- **Rate limiting is DDoS protection**: Authentication storms are CPU attacks. Rate limiting at the network level (iptables/Cloudflare) prevents CPU saturation
- **Audit logs need alerting**: 500,000 log entries is not "noise" — it's a signal that failed alerting thresholds. If auth.log spikes aren't paging someone, you're blind
- **Bastion hosts are critical infrastructure**: A single compromised or unavailable bastion blocks access to entire cluster fleets. Treat it with the same security rigor as a production service

---

## Summary

The attack chain:

```
Attacker scans Shodan for exposed SSH servers
→ Finds bastion host on default port 22
→ Launches distributed brute force: 1,200 IPs, 500K attempts
→ CPU pegged at 100% processing auth requests
→ Legitimate admins locked out
→ 45-minute cluster access outage
→ Emergency IP whitelist restored access
→ fail2ban + SSH hardening + Cloudflare Spectrum deployed
```

Recovery time: 45 minutes. Fixes: 3 hours. The attack was unsophisticated — 500,000 dictionary attempts from rented botnet IPs. But it worked because the server was configured with the defaults from 18 months ago. Security debt accumulates. Review your SSH config today.
