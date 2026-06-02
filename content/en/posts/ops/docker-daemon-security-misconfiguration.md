---
title: "Docker Daemon Misconfiguration — How Exposed Docker Socket Led to Host Takeover"
date: 2026-06-02
weight: 100260
slug: "docker-daemon-security-misconfiguration"
tags: ["security", "docker", "container", "linux", "troubleshooting"]
categories: ["Security"]
description: "A Docker daemon security incident — how mounting the Docker socket into a container allowed an attacker to escape and take over the host, plus all containers on it"
keywords: "docker socket exposed, docker privilege escalation, /var/run/docker.sock security, docker container escape host, docker daemon tcp exposure"
draft: false
featured: true
cover:
  image: ""
  caption: "Docker Daemon Misconfiguration — Incident Response"
---

# Docker Daemon Misconfiguration — How Exposed Docker Socket Led to Host Takeover

## Common Search Queries

- docker.sock exposed container escape
- docker privilege escalation host takeover
- /var/run/docker.sock security
- docker daemon tcp exposed
- docker-in-docker security risk

---

## The Incident

**Environment**: Single Docker host (8C/32G, Ubuntu 22.04), running 15 microservice containers for a CI/CD build farm. Docker API exposed on TCP port 2375 (no TLS).

**Time**: Tuesday 14:30. Security scan alerted: unknown container `evil-miner` running on the CI build host.

**Initial Discovery**: A routine security scan with Falco flagged a container named `evil-miner` running a known cryptominer binary.

```bash
docker ps
CONTAINER ID   IMAGE              COMMAND                  STATUS          NAMES
a1b2c3d4e5f6   xmrig/xmrig        "./xmrig --config=..."   Up 2 hours      evil-miner
f6e5d4c3b2a1   nginx:latest       "nginx -g daemon off;"   Up 14 hours     ci-web
...
```

**Impact**: Host and all 15 build containers compromised. CI/CD credentials, SSH keys, and source code exfiltrated. Host rebuilt from scratch.

---

## Background

The CI/CD build farm ran on a single Docker host. To enable "Docker-in-Docker" builds, the team had mounted the Docker socket (`/var/run/docker.sock`) into the CI runner container. This was a well-known pattern — but also the exact attack vector exploited here.

```bash
# How the CI runner container was started
docker run -d \
  --name ci-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \  # ← Docker socket mounted
  -v /home/ci/workspace:/workspace \
  gitlab-runner:latest
```

Mounting the Docker socket gives the container **root-level access to the Docker daemon**. From inside the container, you can run any Docker command — including launching new containers with full host access.

Additionally, the Docker daemon itself was listening on TCP port 2375 without TLS, configured for "convenience" so the team could connect remotely:

```bash
# /etc/docker/daemon.json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"],
  "tls": false
}
```

An attacker scanning the internal network found port 2375 open. From there, they had full control of the Docker daemon.

---

## Investigation

### Step 1: Check How the Attacker Got In

```bash
# Check Docker daemon logs for API calls
journalctl -u docker.service --since "24 hours ago" | grep -i "POST\|DELETE\|create"
```

The logs showed Docker API calls from two sources:
1. Internal IP `10.0.1.50` — the CI runner container (via socket mount)
2. Internal IP `10.0.1.200` — unknown, via TCP API on port 2375

The attacker found the exposed TCP port via internal network scanning.

### Step 2: Trace the Attacker's Actions

```bash
# Check what containers were created
docker logs docker_audit 2>/dev/null || journalctl -u docker.service | grep -i "create\|pull"
```

The attacker pulled the `xmrig/xmrig` image and started a cryptominer container. But before that, they:

1. Listed all running containers → found the CI runner with socket mount
2. Used the CI runner's Docker socket to launch a privileged container
3. Mounted the host filesystem from the privileged container
4. Exfiltrated SSH keys from `/home/ci/.ssh/`
5. Exfiltrated CI/CD credentials from `/home/ci/workspace/.env`
6. Started the cryptominer as cover

### Step 3: Assess the Damage

```bash
# Check mounted volumes from the attacker's container
docker inspect evil-miner | jq '.[].Mounts'

# Check if host SSH keys were accessed
grep -rn "BEGIN OPENSSH PRIVATE KEY" /home/ci/.ssh/
```

All SSH keys, CI/CD tokens, and environment variables on the host were compromised. If this host had access to production clusters, those credentials would need rotation too.

---

## Root Cause

1. **Docker socket mounted into containers**: The CI runner had `docker.sock` mounted, giving any process in that container root-level Docker daemon access
2. **Docker TCP API exposed without TLS**: Port 2375 was open on `0.0.0.0` with no authentication or encryption
3. **No firewall on Docker API port**: No iptables rule or security group restricted access to port 2375
4. **Privileged containers allowed**: No policy preventing creation of privileged containers via the API

---

## Resolution

### Emergency (Immediate)

```bash
# Stop all unknown containers
docker stop evil-miner
docker rm evil-miner

# Audit and stop any other suspicious containers
docker ps -a | grep -v -f known-containers.txt | awk '{print $1}' | xargs docker rm -f

# Immediately disable TCP API access
systemctl stop docker
# Remove tcp://0.0.0.0:2375 from daemon.json
sed -i 's|"tcp://0.0.0.0:2375",||' /etc/docker/daemon.json
systemctl start docker

# Block port 2375 at firewall
iptables -A INPUT -p tcp --dport 2375 -j DROP
ufw deny 2375
```

### Harden Docker Configuration

```bash
# /etc/docker/daemon.json — secure configuration
{
  "hosts": ["unix:///var/run/docker.sock"],
  "tls": true,
  "icc": false,
  "live-restore": true,
  "userland-proxy": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Never Mount Docker Socket in Containers

Instead of mounting the Docker socket, use:

1. **Docker-in-Docker (dind)**: Run a separate Docker daemon inside the container
2. **Rootless Docker**: Run Docker without root in the CI container
3. **Podman**: Rootless container engine for CI/CD (no daemon needed)

```yaml
# Alternative: use dind (Docker-in-Docker)
services:
  - docker:dind

script:
  - docker build -t myapp .
  - docker run myapp
```

If Docker socket mount is **absolutely necessary**:

1. Use read-only mount: `-v /var/run/docker.sock:/var/run/docker.sock:ro`
2. Use Docker API proxy with authentication (e.g., `bobcob/nginx-docker-proxy`)
3. Restrict which images can be pulled via registry allowlisting

### Secure Docker API

```bash
# If TCP access is necessary, use TLS certificates
# Generate CA and client certificates
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=docker-host" -sha256 -new -key server-key.pem -out server.csr

# Configure daemon with TLS
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "tlsverify": true
}
```

### Monitoring and Audit

```bash
# Docker audit logging
dockerd --authorization-plugin=audit-plugin
# Or use Falco rules for Docker events
```

```yaml
# Falco rule: detect Docker socket mounts
- rule: Docker Socket Mounted in Container
  desc: Detect containers mounting the Docker socket
  condition: container.mount contains "/var/run/docker.sock"
  output: "Docker socket mounted in container (user=%user.name container=%container.name)"
  priority: CRITICAL
```

```yaml
# Prometheus alert for exposed Docker API
groups:
- name: docker-security
  rules:
  - alert: DockerAPIAccessible
    expr: probe_success{target="tcp://docker-host:2375"} == 1
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Docker API is accessible on port 2375 without TLS"
```

---

## Lessons Learned

- **`/var/run/docker.sock` is root-equivalent**: Mounting the Docker socket in a container gives that container root access to the host. There is no sandbox, no isolation — it's a root backdoor
- **Docker TCP API is root access without password**: Exposing port 2375 without TLS is like putting the root password on a public website. Anyone who finds it owns the host
- **CI/CD runners should never have socket access**: Use dind, rootless Docker, or Podman instead. Build containers don't need daemon access
- **Docker-in-Docker patterns are dangerous defaults**: The convenience of socket mounting has caused countless breaches. It's not a CI/CD requirement — it's a security risk
- **Audit container mounts regularly**: `docker inspect` shows all mounts. Scan for `docker.sock` mounts weekly

---

## Summary

The attack chain:

```
Docker host configured with TCP API on port 2375 (no TLS)
→ Attacker scans internal network, finds open port
→ Connects to Docker API — no password needed
→ Lists containers, finds CI runner with docker.sock mounted
→ Launches privileged container with host filesystem mount
→ Exfiltrates SSH keys and CI/CD credentials
→ Starts cryptominer container as cover
→ 15 build containers compromised, host fully owned
```

Recovery: host rebuild + credential rotation, 6 hours. Fix: 30 minutes of daemon reconfiguration. The Docker socket is powerful — treat it like a root password. Because that's exactly what it is.
