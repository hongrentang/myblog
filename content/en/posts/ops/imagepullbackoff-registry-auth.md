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

# ImagePullBackOff Troubleshooting: Expired Registry Credentials

## Common Search Queries

If you're landing here from a search, this post covers:

- k8s imagepullbackoff fix
- pod failed to pull image ErrImagePull
- kubernetes imagePullSecrets not working
- docker registry authentication failed kubernetes
- harbor registry pull secret expired

---

## The Incident

**Environment**: K8S v1.28, containerd 1.7, Harbor private registry, regular cluster maintenance.

**Time**: Thursday 10:00, deployment window.

**Symptoms**: After the CI/CD pipeline completed the new release, all new pods remained in `ImagePullBackOff` status.

```bash
NAMESPACE    NAME                              READY   STATUS              RESTARTS   AGE
production   order-service-v2-7d8f9d4c5f-a1b2  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-c3d4  0/1     ImagePullBackOff    0          3m
production   order-service-v2-7d8f9d4c5f-e5f6  0/1     ImagePullBackOff    0          3m
```

**Impact**: New version rollout failed. Rolled back to the previous version. But the old version's image was still in the registry — why couldn't the new one be pulled?

---

## Misdiagnosis

### First Reaction: Image Tag Missing

Seeing `ImagePullBackOff`, I immediately suspected the CI pipeline failed to push the image. I checked Harbor — the image was there with the correct tag.

### Next Suspect: imagePullPolicy

Checked the Deployment config. `imagePullPolicy: IfNotPresent` — reasonable. It should use a cached image if available. But the problem persisted.

### Manual Investigation

```bash
kubectl describe pod order-service-v2-7d8f9d4c5f-a1b2 -n production
```

Events output:

```
Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       4m    default-scheduler  Successfully assigned...
  Normal   Pulling         4m    kubelet            Pulling image "harbor.internal/library/order-service:v2.1.0"
  Warning  Failed          4m    kubelet            Failed to pull image "harbor.internal/library/order-service:v2.1.0": rpc error: code = Unknown desc = failed to pull and unpack image: unauthorized: authentication required
  Warning  Failed          4m    kubelet            Error: ErrImagePull
  Warning  Failed          3m    kubelet            Error: ImagePullBackOff
```

Key detail: **unauthorized: authentication required** — the image exists, but the cluster can't authenticate to pull it.

---

## Investigation

### Step 1: Check imagePullSecrets

```bash
kubectl get pod order-service-v2-7d8f9d4c5f-a1b2 -n production -o yaml | grep imagePullSecrets -A 5
```

```
imagePullSecrets:
- name: harbor-registry-cred
```

The Secret exists. Let's check its contents:

```bash
kubectl get secret harbor-registry-cred -n production -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
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

Decoding `auth`: `echo 'YWRtaW46SGFyYm9yMTIzNDU=' | base64 -d` → `admin:Harbor12345`

`Harbor12345` — the default Harbor installation password, which was rotated three months ago.

### Step 2: Verify Network Connectivity

```bash
# Test from a cluster node
curl -I https://harbor.internal/v2/
```

```
HTTP/2 401 
```

401 is expected without credentials — the point is the network path is working. Harbor is reachable.

### Step 3: Trace the Secret Origin

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-registry-cred
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>
```

The Git history showed this Secret was created during the initial cluster setup — three months ago. It had never been updated. Harbor's password was rotated in the meantime, but the K8S Secret was never synced.

---

## Root Cause

1. Security policy required quarterly Harbor password rotation
2. The ops team rotated the password on Harbor, but **forgot to update the corresponding K8S imagePullSecrets**
3. Existing pods continued running because the node had cached images from the initial pull
4. The new release required a fresh pull on nodes without cache — immediately exposing the credential mismatch

**Biggest misdiagnosis**: I spent the first 5 minutes checking CI pipeline logs, convinced the image wasn't pushed. The image was fine — the cluster simply couldn't pull it.

---

## Recovery

### Hotfix (2 minutes)

Update the Secret with the new Harbor password:

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=admin \
  --docker-password='NewHarbor!Pass2024' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Force pod recreation:

```bash
kubectl delete pod -n production -l app=order-service --force
```

New pods pulled the image successfully. Service restored.

---

## Long-Term Fixes

### 1. Use Robot Accounts

Never use admin credentials for image pulling. Create read-only robot accounts:

```bash
kubectl create secret docker-registry harbor-registry-cred \
  --namespace=production \
  --docker-server=harbor.internal \
  --docker-username=robot$puller \
  --docker-password='<robot-token>'
```

### 2. Auto-Sync Secrets with External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-registry-cred
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: harbor-registry-cred
    creationPolicy: Owner
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secrets/harbor
      property: dockerconfigjson
```

### 3. Pre-Deployment Image Pull Check

Add a validation step in CI/CD:

```bash
kubectl run image-pull-test --image=harbor.internal/library/order-service:v2.1.0 \
  --restart=Never --image-pull-policy=Always --dry-run=client -o yaml | kubectl apply -f -

sleep 10
STATUS=$(kubectl get pod image-pull-test -o jsonpath='{.status.phase}')
if [ "$STATUS" != "Running" ]; then
  echo "Image pull validation failed!"
  kubectl describe pod image-pull-test
  exit 1
fi
```

### 4. Monitor ImagePullBackOff Events

Prometheus alerting rule:

```yaml
groups:
- name: image-pull-alerts
  rules:
  - alert: ImagePullBackOff
    expr: kube_pod_status_phase{phase="Pending"} > 0
    for: 2m
    annotations:
      summary: "Pod {{ $labels.pod }} is in ImagePullBackOff state"
```

---

## Quick Reference

```bash
# Check pod events
kubectl describe pod <pod> -n <namespace>

# Decode image pull secret
kubectl get secret <secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Test image pull
kubectl run test-pod --image=<image> --restart=Never

# Update registry credentials
kubectl create secret docker-registry <name> \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<pass> \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Summary

Investigation chain:

```
New release failed
  → Pod ImagePullBackOff
  → Misdiagnosed: CI didn't push image
  → Checked Harbor (image exists)
  → describe Pod (unauthorized)
  → Inspected Secret (expired credentials)
  → Updated Secret → Pod recovered
```

Total time from alert to recovery: 8 minutes.

**Lesson**: Image pull credentials are a ticking time bomb. Every credential rotation at the registry must be synced to K8S. Use External Secrets Operator to automate this — human memory is not a reliable synchronization mechanism.
