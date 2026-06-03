---
title: "etcd Unencrypted Data Exposure — How a Backup File Leaked the Entire Cluster's Secrets"
date: 2026-06-03
weight: 100300
slug: "etcd-unencrypted-data-exposure"
tags: ["kubernetes", "security", "etcd", "encryption", "troubleshooting"]
categories: ["Security"]
description: "An etcd data exposure incident — how an unencrypted etcd snapshot file stored on a backup server leaked cluster secrets, service account tokens, and ConfigMap data to an unauthorized party"
keywords: "etcd encryption at rest, kubernetes etcd backup security, etcd snapshot unencrypted, k8s secret encryption, etcd data breach"
draft: false
featured: true
cover:
  image: ""
  caption: "etcd Unencrypted Data Exposure — Incident Response"
---

# etcd Unencrypted Data Exposure — How a Backup File Leaked the Entire Cluster's Secrets

## Common Search Queries

- etcd encryption at rest kubernetes
- etcd backup exposed secrets
- kubernetes etcd snapshot security
- k8s secret encryption etcd
- etcd data breach incident

---

## The Incident

**Environment**: K8S v1.27, 3-node etcd cluster, single-region production, backup stored on a shared NAS.

**Time**: Wednesday 14:00. Security team received an alert from an external security researcher: an etcd snapshot file was accessible on a publicly exposed NAS share.

**Initial Discovery**: A security researcher performing routine Shodan scanning found an NFS share with no authentication. Inside: `etcd-snapshot-2026-06-02.db`. They downloaded it, decoded it, found Kubernetes secrets, and responsibly disclosed.

```bash
# The researcher could mount the NFS share without credentials
mount -t nfs 203.0.113.50:/backups /mnt/backups
ls /mnt/backups/
etcd-snapshot-2026-06-01.db
etcd-snapshot-2026-06-02.db
kubernetes-manifests/
```

**Impact**: Every secret in the Kubernetes cluster — TLS keys, database passwords, API tokens, service account credentials — was exposed to anyone who found that NFS share. Estimated exposure window: 3-6 months.

---

## Background

etcd is the backing store for Kubernetes. Every cluster resource — Secrets, ConfigMaps, ServiceAccounts, RBAC policies — is stored in etcd. If someone gets access to an etcd snapshot, they have access to everything in the cluster.

The team had automated etcd backups running daily via a cronjob:

```bash
# /etc/cron.d/etcd-backup
0 2 * * * root ETCDCTL_API=3 etcdctl snapshot save \
  /backups/etcd-snapshot-$(date +\%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

The backups were written to a NAS share mounted at `/backups`. The NAS was supposed to be internal-only, but a misconfigured firewall rule had exposed the NFS service to the internet.

The bigger issue: **etcd encryption at rest was not enabled**. All data in etcd was stored in plaintext:

```bash
# Check if encryption at rest is enabled
kubectl get EncryptionConfiguration -o yaml 2>/dev/null || echo "No encryption config found"
```

Output: `No encryption config found`. Every resource in etcd — including Secrets — was stored as plaintext JSON or YAML.

```bash
# Anyone with the snapshot could decode any secret
strings etcd-snapshot-2026-06-02.db | grep -A2 "kind: Secret" | head -20
```

---

## Investigation

### Step 1: Confirm the Exposure

```bash
# Check if encryption at rest is configured
kubectl describe encryptionconfig 2>/dev/null || echo "No EncryptionConfig resource exists"
```

There was no `EncryptionConfiguration` in the cluster. All etcd data was stored unencrypted.

### Step 2: Identify the Exposed Data Scope

```bash
# What would the attacker have found in the snapshot?
ETCDCTL_API=3 etcdctl get /registry/secrets/production/db-credentials \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Every Secret, ConfigMap, ServiceAccount token, and TLS certificate in the cluster was exposed.

### Step 3: Check Access Logs

```bash
# Check NAS access logs for unauthorized access
grep -i "access|mount|read" /var/log/nas/audit.log | grep -v "10.0.0." | head -20
```

The NAS logs showed connections from IPs outside the internal network range — confirmation that the exposure had been discovered by unauthorized parties.

---

## Root Cause

1. **Encryption at rest not enabled**: etcd stored all data in plaintext. No encryption was configured for Kubernetes Secrets
2. **Backup files unencrypted**: The etcd snapshot was stored without any encryption on the NAS
3. **NAS accidentally exposed**: A misconfigured firewall rule made the NFS share accessible from the internet
4. **No backup access monitoring**: No alerting for NFS connections from external IPs
5. **No backup integrity verification**: The backup cronjob ran but nobody verified the backups were secure

---

## Resolution

### Emergency (Immediate)

```bash
# 1. Block external access to NAS
iptables -A INPUT -s 0.0.0.0/0 -p tcp --dport 2049 -j DROP
iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 2049 -j ACCEPT

# 2. Rotate ALL cluster secrets
# (Since the etcd snapshot contained every secret)
kubectl delete secret --all-namespaces --all
# Force recreation of service account tokens
kubectl delete pod --all-namespaces --all

# 3. Assume all exposed credentials are compromised
# - Rotate database passwords
# - Reissue TLS certificates
# - Regenerate cloud provider credentials
```

### Enable Encryption at Rest for etcd

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

```bash
# Generate the encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# Apply the configuration
kubectl apply -f encryption-config.yaml

# Move the encryption config into place on the control plane
# Edit kube-apiserver manifest to add:
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Encrypt Existing Data

```bash
# After enabling encryption, existing data in etcd is still unencrypted
# Rewrite all secrets to encrypt them
kubectl get secrets --all-namespaces -o json | jq -r '.items[].metadata.name' | while read secret; do
  kubectl get secret $secret -n $ns -o json | kubectl replace -f -
done
```

### Secure etcd Backups

```bash
# Encrypt backup files with GPG
etcdctl snapshot save /tmp/etcd-snapshot.db
gpg --encrypt --recipient admin@company.com /tmp/etcd-snapshot.db
mv /tmp/etcd-snapshot.db.gpg /backups/

# Or use a dedicated backup tool with built-in encryption
# velero install --use-volume-snapshots=false --encryption-key-file=/path/to/key
```

### Secure NAS Access

```bash
# Restrict NFS to internal IPs only
# /etc/exports
/backups 10.0.0.0/8(rw,no_root_squash,no_subtree_check)

# Add authentication
# Use Kerberized NFS or at minimum export with IP restrictions
```

### Monitoring

```yaml
# Alert on NFS access from unexpected IPs
- alert: NFSExternalAccess
  expr: nfs_mount_count{src_ip!~"10.0.0.0/8"} > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "NFS mount from external IP detected"
```

---

## Lessons Learned

- **etcd stores everything in plaintext by default**: Kubernetes Secrets in etcd are base64-encoded but NOT encrypted. Anyone with etcd access has all secrets
- **Encryption at rest is not optional**: It's a checkbox in cluster bootstrap for a reason. Without it, physical access to etcd data = complete cluster compromise
- **Backup files are secrets too**: An etcd snapshot contains the entire cluster state. Encrypt backups before storing them anywhere
- **NAS is not a security boundary**: NFS shares are often exposed wider than expected. Verify firewall rules regularly
- **Assume all backups are public**: Build your security model around the assumption that backup files will eventually leak. Encryption at rest + encrypted backups = defense in depth

---

## Summary

The attack chain:

```
etcd configured without encryption at rest
→ Daily cronjob creates etcd snapshot on NAS
→ NAS NFS share accidentally exposed to internet (firewall change)
→ Security researcher finds NFS share via Shodan
→ Downloads etcd snapshot
→ Decodes all Secrets, ConfigMaps, ServiceAccount tokens, TLS certs
→ Responsible disclosure to security team
```

Recovery: full credential rotation, 6 hours. Fix: encryption at rest + encrypted backups, 1 hour. Prevention: 10 minutes during cluster bootstrap. etcd is the heartbeat of Kubernetes — encrypt it like one.
