---
title: "ImagePullBackOff Troubleshooting: Expired Registry Credentials Blocking Deployment"
date: 2026-05-19
weight: 100030
slug: "k8s-imagepullbackoff-registry-auth"
tags: ["kubernetes", "troubleshooting"]
categories: ["K8S"]
description: "Full walkthrough of diagnosing ImagePullBackOff — from misdiagnosing CI pipeline failure to finding expired registry credentials"
keywords: "k8s imagepullbackoff fix, pod image pull failed, ErrImagePull fix, kubernetes registry auth, imagePullSecrets troubleshooting"
draft: false
featured: true
cover:
  image: "/images/imagepullbackoff-banner.svg"
  caption: "ImagePullBackOff Troubleshooting & Diagnostics"
---

# ImagePullBackOff Troubleshooting: Expired Registry Credentials Blocking Deployment

## Common Search Queries

If you're landing here from a search, this post covers:

- k8s imagepullbackoff fix
- pod failed to pull image ErrImagePull
- kubernetes imagePullSecrets not working
- docker registry authentication failed kubernetes
- harbor registry pull secret expired
- containerd image pull troubleshooting
- K8S registry credential rotation failed

---

## The Incident

**Environment**: K8S v1.28.5, containerd 1.7.8, Calico CNI v3.26, Harbor v2.8 private registry, 12-node cluster (3 Master + 9 Worker), 200+ microservices.

**Time**: Thursday 10:00 AM, regular deployment window. Harbor quarterly password rotation was completed the day before.

**Symptoms**: After the CI/CD pipeline finished building and pushing the new version, monitoring fired — all new order service pods were stuck in `ImagePullBackOff`. The QA team also reported they couldn't deploy the latest build.

```bash
NAMESPACE    NAME                              READY   STATUS              RESTARTS   AGE
production   order-service-v2-7d8f9d4c5f-a1b2  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-c3d4  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-e5f6  0/1     ImagePullBackOff    0          3m
staging      order-service-v2-6a8b7c9d0e-f1g2  0/1     ImagePullBackOff    0          5m
```

**Impact**: New version rollout completely blocked. Rolled back to the previous version, but new features couldn't be delivered. QA environment was also blocked, stalling the entire sprint's testing progress. The old version continued running fine — which meant the problem was specifically with **pulling the new image**, not with the cluster itself.

---

## Misdiagnosis

### First Reaction: CI Didn't Push the Image

When you see `ImagePullBackOff`, the most natural thought is "the image wasn't pushed." I immediately ran three checks:

```bash
# 1. Verify the artifact exists in Harbor
curl -u admin:password "https://harbor.internal/api/v2.0/projects/library/repositories/order-service/artifacts?v=2.1.0" | jq .

# 2. Try pulling directly with Docker
docker pull harbor.internal/library/order-service:v2.1.0

# 3. Check CI pipeline logs for the push step
# Search for "push" and "success" in Jenkins console output
```

Results: The image was there with the correct tag. CI logs showed the push succeeded. The Harbor UI even showed all layers with correct digests and sizes.

### Next Suspect: imagePullPolicy

I pulled the Deployment YAML to check image pull configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v2
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: order-service
        image: harbor.internal/library/order-service:v2.1.0
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor-registry-cred
```

`imagePullPolicy: IfNotPresent` means if the node already has an image with that tag, kubelet uses the cached version. But since the new `v2.1.0` tag was freshly built, no node would have it cached — so this policy wasn't helping us skip the pull.

### Even Suspected containerd Configuration

Since the cluster uses containerd rather than Docker, I briefly wondered if the containerd registry mirror configuration was wrong:

```bash
# Check containerd config for Harbor mirror settings
cat /etc/containerd/config.toml | grep -A 10 "harbor"
```

There was no Harbor mirror configuration at all — containerd would connect directly via HTTPS. That path was clean.

---

## Investigation

### Step 1: Examine Pod Events

`kubectl describe pod` is always the first step for any pod issue. Events tell you the kubelet's perspective directly, which is faster than digging through logs.

```bash
kubectl describe pod order-service-v2-7d8f9d4c5f-a1b2 -n production
```

Relevant output:

```
Name:             order-service-v2-7d8f9d4c5f-a1b2
Namespace:        production
Priority:         0
Service Account:  order-service
Node:             worker-04/10.0.1.44
Labels:           app=order-service
                  version=v2.1.0
Status:           Pending
IP:               
IPs:              <none>

Conditions:
  Type           Status
  PodScheduled   True 
  Initialized    False
  Ready          False
  ContainersReady False

Containers:
  order-service:
    Container ID:   
    Image:          harbor.internal/library/order-service:v2.1.0
    Image ID:       
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0

Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       4m    default-scheduler  Successfully assigned production/order-service-v2-7d8f9d4c5f-a1b2 to worker-04
  Normal   Pulling         3m    kubelet            Pulling image "harbor.internal/library/order-service:v2.1.0"
  Warning  Failed          3m    kubelet            Failed to pull image "harbor.internal/library/order-service:v2.1.0": rpc error: code = Unknown desc = failed to pull and unpack image: unauthorized: authentication required
  Warning  Failed          3m    kubelet            Error: ErrImagePull
  Warning  Failed          2m    kubelet            Error: ImagePullBackOff
  Normal   BackOff         2m    kubelet            Back-off pulling image "harbor.internal/library/order-service:v2.1.0"
```

The key detail: **unauthorized: authentication required**. Not a missing image, not a network issue — an **authentication failure**.

Pay attention to the event timeline:
1. `Scheduled` — Pod assigned to worker-04
2. `Pulling` — kubelet started pulling
3. `Failed` → `ErrImagePull` → `ImagePullBackOff` — failure and backoff
4. `BackOff` — kubelet enters exponential backoff, starting at 5s and increasing to 5min max

### Step 2: Check imagePullSecrets

The Pod's `imagePullSecrets` field specifies which credentials to use for image pulling. Let's verify they're configured:

```bash
kubectl get pod order-service-v2-7d8f9d4c5f-a1b2 -n production -o yaml | grep -A 5 "imagePullSecrets"
```

```
imagePullSecrets:
- name: harbor-registry-cred
```

The Secret exists. But what's actually in it?

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

```json
{
  "auths": {
    "harbor.internal": {
      "auth": "YWRtaW46SGFyYm9yMTIzNDU="
    }
  }
}
```

Decode the `auth` field:

```bash
echo 'YWRtaW46SGFyYm9yMTIzNDU=' | base64 -d
# Output: admin:Harbor12345
```

`Harbor12345` — the **default Harbor installation password**. This password was rotated three months ago during a security audit, but the K8S Secret was never updated.

### Step 3: Verify Registry Connectivity

To completely rule out network-level issues, I ran a comprehensive check from a cluster node:

```bash
# Verify DNS resolution
nslookup harbor.internal

# Verify TCP connectivity with timing
time nc -zv harbor.internal 443

# Verify TLS certificate
openssl s_client -connect harbor.internal:443 -servername harbor.internal < /dev/null 2>/dev/null | openssl x509 -noout -dates

# Test with old credentials
echo -n 'admin:Harbor12345' | base64
curl -H "Authorization: Basic YWRtaW46SGFyYm9yMTIzNDU=" https://harbor.internal/v2/_catalog
```

Output:

```
;; ANSWER SECTION:
harbor.internal.  60  IN  A  10.0.1.200

Connection to harbor.internal port 443 succeeded!

notBefore=Mar 15 00:00:00 2026 GMT
notAfter=Jun 13 00:00:00 2026 GMT

{"errors":[{"code":"UNAUTHORIZED","message":"authentication required"}]}
```

DNS resolved, TCP connected, TLS certificate valid, old credentials rejected. Confirmed: the problem is solely the credentials.

Test with the new password:

```bash
echo -n 'admin:NewHarbor!Pass2024' | base64
curl -H "Authorization: Basic $(echo -n 'admin:NewHarbor!Pass2024' | base64)" https://harbor.internal/v2/_catalog
```

The new password worked — returned the repository list successfully.

### Step 4: Trace the Secret's Lifecycle

Using Git history to find when this Secret was created:

```bash
git log --all --oneline -- "*/harbor-registry-cred*"
git show <commit-hash>
```

```
commit a3f8c2d (initial-cluster-setup)
Date:   Mon Feb 10 14:30:00 2026

    Add initial Kubernetes manifests for production namespace
    
    - Includes harbor-registry-cred Secret
    - Default Harbor password used
```

The Secret was created during initial cluster setup — 3 months ago — using the default password. Harbor had its quarterly password rotation since then, but the Secret was never updated. The ops runbook simply didn't include "sync K8S imagePullSecrets" as a step in the password rotation procedure.

---

## Root Cause Analysis

1. **Security compliance**: Security policy required quarterly Harbor admin password rotation. The ops team executed the rotation on schedule
2. **Process gap**: After updating the password in Harbor, **no one updated the corresponding K8S imagePullSecrets**. This was a process failure, not a technical one
3. **Cache masked the problem**: Old pods (with old image tags) had cached images on nodes. Since kubelet only pulls when there's no cache, the expired Secret went unnoticed for 3 months
4. **New tag triggered the failure**: The new CI build produced a fresh image tag that no node had cached. Kubelet had to pull from Harbor, immediately exposing the credential mismatch

**Critical insight**: K8S only pulls images during pod scheduling, and only performs an actual pull when no cached image exists. "It's running" does NOT mean "credentials are valid." This is an easy trap to fall into.

**Biggest misdiagnosis**: I spent the first 5 minutes glued to the CI pipeline logs, repeatedly verifying the build and push steps. The image was perfectly fine in Harbor — the problem was entirely on the pull side, completely unrelated to CI.

---

## Recovery

### Hotfix (2 minutes)

Update the Secret with the new password. Using the `kubectl create secret --dry-run=client` technique for quick in-place updates without touching YAML files:

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=admin \
  --docker-password='NewHarbor!Pass2024' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify the updated Secret:

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

```json
{
  "auths": {
    "harbor.internal": {
      "auth": "YWRtaW46TmV3SGFyYm9yIVBhc3MyMDI0"
    }
  }
}
```

Force pod recreation:

```bash
kubectl delete pod -n production -l app=order-service --force
```

Watch the new pods come up:

```bash
kubectl get pods -n production -l app=order-service -w
```

```
NAME                              READY   STATUS              RESTARTS   AGE
order-service-v2-7d8f9d4c5f-x1y2  0/1     Init:ErrImagePull   0          0s
order-service-v2-7d8f9d4c5f-x1y2  0/1     PodInitializing     0          8s
order-service-v2-7d8f9d4c5f-x1y2  1/1     Running             0          12s
```

Pods transitioned to Running in about 12 seconds. Production restored within 2 minutes.

### Rollback Safety

In case the hotfix introduced unexpected issues, the old version deployment was preserved as a fallback:

```bash
kubectl rollout undo deployment/order-service-v2 -n production
```

---

## Long-Term Fixes

### 1. Use Robot Accounts Instead of Admin

Using an admin account for image pulling is an anti-pattern — excessive permissions (can delete images, modify project config), and sharing credentials across a team magnifies the blast radius of any rotation.

Harbor supports robot accounts with project-scoped, read-only, long-lived tokens:

```bash
# Create robot account in Harbor UI
# Project → Robots → Add Robot Account
# Name: robot-order-service-puller
# Permission: pull-only
# Expiry: never

# Create corresponding K8S Secret
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=robot$order-service-puller \
  --docker-password='<harbor-generated-token>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Benefits of robot accounts:
- Granular permissions (read-only, project-scoped)
- Token is independent of user passwords — password rotations don't affect it
- Harbor audit logs show exactly which service pulled which image

### 2. Auto-Sync Secrets with External Secrets Operator

Manual Secret management is unreliable. Use External Secrets Operator to sync from Vault automatically:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "v1/auth/kubernetes"
          role: "external-secrets"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-registry-cred
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: harbor-registry-cred
    creationPolicy: Owner
    deletionPolicy: Delete
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secrets/harbor
      property: dockerconfigjson
```

When Harbor passwords are rotated, only the Vault value needs updating. ESO syncs to K8S within 1 hour automatically.

### 3. Pre-Deployment Image Pull Check in CI/CD

Add a validation step in the deployment pipeline to catch credential issues before they block production:

```yaml
# GitLab CI example
image-pull-check:
  stage: verify
  script:
    - |
      kubectl run image-pull-test \
        --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG \
        --restart=Never \
        --image-pull-policy=Always \
        --dry-run=client -o yaml | kubectl apply -f -
      
      # Wait up to 30 seconds
      for i in $(seq 1 30); do
        STATUS=$(kubectl get pod image-pull-test -o jsonpath='{.status.phase}')
        if [ "$STATUS" = "Running" ]; then
          echo "✓ Image pull successful"
          kubectl delete pod image-pull-test --force --wait=false
          exit 0
        fi
        sleep 1
      done
      
      # Timeout — print details and fail
      echo "✗ Image pull failed!"
      kubectl describe pod image-pull-test
      kubectl delete pod image-pull-test --force --wait=false
      exit 1
```

### 4. Prometheus Alerting for ImagePullBackOff

Set up proactive monitoring:

```yaml
groups:
- name: kubernetes-pods
  rules:
  - alert: ImagePullBackOff
    expr: kube_pod_status_phase{phase="Pending"} > 0
    for: 3m
    labels:
      severity: critical
    annotations:
      summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} stuck in ImagePullBackOff"
      description: "Pod has been unable to pull image for over 3 minutes. Check imagePullSecrets and registry accessibility."
      runbook: "https://internal.runbooks/imagepullbackoff"

  - alert: RegistryAuthExpiring
    expr: time() - secret_created_timestamp{type="kubernetes.io/dockerconfigjson"} > 86400 * 80
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Docker config secret {{ $labels.secret }} is 80+ days old"
      description: "Registry credentials may expire soon. Verify and rotate if needed."
```

### 5. Version Secrets in GitOps

Store the baseline Secret definition in Git and manage it through ArgoCD or Flux:

```yaml
# gitops/infrastructure/harbor-registry-cred.yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-registry-cred
  namespace: production
  annotations:
    sealedsecrets.bitnami.com/managed: "true"
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <encrypted-by-sealed-secrets>
```

Changes go through Merge Request with approval, providing an audit trail.

### 6. Regular Credential Validation Script

Periodically verify that registry credentials are still valid:

```bash
#!/bin/bash
# check-registry-creds.sh
# Periodically verify all dockerconfigjson secrets across the cluster

for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  for secret in $(kubectl get secret -n $ns --field-selector type=kubernetes.io/dockerconfigjson -o name 2>/dev/null); do
    echo "Checking $secret in $ns"
    REGISTRY=$(kubectl get $secret -n $ns -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[0]')
    AUTH=$(kubectl get $secret -n $ns -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths[].auth')
    
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Basic $AUTH" https://$REGISTRY/v2/)
    if [ "$HTTP_CODE" = "401" ]; then
      echo "WARNING: Secret $secret in $ns has invalid credentials!"
    fi
  done
done
```

---

## Reproduction Steps

To reproduce this scenario in a test environment:

1. Change the Harbor admin password
2. Do NOT update the K8S imagePullSecrets
3. Schedule a new pod on a node without cached images:
   ```bash
   kubectl cordon <node-with-cache>
   kubectl delete pod <existing-pod>
   ```
   Or deploy to a fresh node (e.g., scale up a new node group)
4. Observe ImagePullBackOff

---

## Quick Reference

```bash
# Check pod status and events (always step one)
kubectl describe pod <pod> -n <namespace>

# Check which imagePullSecrets the pod uses
kubectl get pod <pod> -n <namespace> -o yaml | grep -A 5 "imagePullSecrets"

# Decode registry credentials
kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .

# Test if credentials are valid
AUTH=$(kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths[].auth')
curl -H "Authorization: Basic $AUTH" https://<registry>/v2/_catalog

# Update registry secret in-place
kubectl create secret docker-registry <name> \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  --dry-run=client -o yaml | kubectl apply -f -

# Test image pull directly
kubectl run test-pod --image=<image> --restart=Never --image-pull-policy=Always

# List all dockerconfigjson secrets across all namespaces
kubectl get secret --all-namespaces --field-selector type=kubernetes.io/dockerconfigjson

# Check node image cache
crictl images | grep <image-name>
```

---

## Common ImagePullBackOff Causes

Beyond expired credentials, ImagePullBackOff can be caused by several other issues:

| Cause | Error Signature | Investigation Path |
|-------|----------------|-------------------|
| Expired credentials | `unauthorized` | Check Secret contents |
| Image not found | `not found` / `manifest unknown` | Verify CI push and registry |
| Registry rate limiting | `rate limit` / `too many requests` | Check registry rate config |
| Node disk full | `no space left on device` | Check node disk usage |
| containerd error | `failed to unpack` | Check containerd logs |
| Network policy blocking | `i/o timeout` / `connection refused` | Check network policies |
| Unsupported image format | `unsupported manifest format` | Check image architecture |

---

## Summary

Investigation chain:

```
New release failed
  → Pod ImagePullBackOff
  → Misdiagnosed: CI didn't push image (checked Harbor — image exists)
  → describe Pod → unauthorized
  → Inspected Secret (admin:Harbor12345 — default password)
  → Harbor password rotated but Secret not updated
  → Updated Secret → Rebuilt Pod → Recovered
```

Total time from alert to recovery: 8 minutes. First 5 minutes were wasted on the wrong hypothesis.

**Key Lessons**:

1. **Expired credentials are a time bomb**: As long as nodes have cached images, an expired Secret produces zero symptoms. It only surfaces during the next deployment
2. **Never use admin accounts for image pulling**: Robot accounts with scoped permissions are the standard practice
3. **Automate, don't rely on manual steps**: External Secrets Operator / Sealed Secrets eliminates human forgetfulness
4. **kubectl describe pod is the critical first step**: The Events field shows kubelet's perspective directly — don't skip it

One more thing: `ImagePullBackOff` uses exponential backoff. The first retry happens after 5s, then 10s, 20s, 40s... up to a 5-minute max. If the issue is urgent (like blocking production), manually deleting the pod resets the backoff timer, triggering an immediate retry instead of waiting.
