---
title: "PVC Mount Failure — When Persistent Volume Claims Refused to Attach and Pods Were Stuck Pending"
date: 2026-06-10
weight: 100480
slug: "pvc-mount-failure-pod-not-scheduling"
tags: ["kubernetes", "storage", "pvc", "troubleshooting", "volume"]
categories: ["Troubleshooting"]
description: "A PVC mount failure incident — how a CSI driver version mismatch and incorrect fsGroup policy caused persistent volume claims to fail mounting, leaving all stateful pods stuck in Pending state"
keywords: "kubernetes pvc mount failure, pod stuck pending pvc, csi driver troubleshooting, pvc attach timeout, kubernetes storage troubleshooting"
draft: false
featured: true
cover:
  image: ""
  caption: "PVC Mount Failure — Troubleshooting"
---

## Common Search Queries

PVC mount failure pod stuck pending, volume attachment timeout kubernetes, CSI driver registration failed, fsGroup policy CSIDriver, pod pending PVC not mounting, kubernetes storage troubleshooting, EBS CSI driver upgrade, volume already used by pod that is not running, node-driver-registrar failure, kubernetes volume attach detach lifecycle.

---

## The Incident

### Environment

| Component | Version / Spec |
|-----------|---------------|
| Kubernetes | v1.29 |
| AWS EBS CSI Driver | v1.20 |
| Stateful Workloads | 20 StatefulSets |
| Storage Class | AWS EBS gp3 (WaitForFirstConsumer, Delete reclaim policy) |
| Node OS | Amazon Linux 2 |
| Container Runtime | containerd 1.7.x |
| Region | AWS ap-southeast-1 |

### Timeline

A routine Kubernetes control plane upgrade from v1.28 to v1.29 was rolled out during a planned maintenance window. The upgrade itself completed without errors — all control plane components (kube-apiserver, kube-controller-manager, kube-scheduler) were running and healthy. Node images were updated in parallel via a managed node group refresh.

However, within minutes of the node group refresh completing, the on-call engineer started receiving alerts:

- Multiple StatefulSet pods stuck in `Pending` state
- Rolling updates of existing StatefulSets would not progress
- New deployments with PVCs failed to schedule

### Symptoms

1. **Pod Status — Stuck Pending**: All newly created pods that referenced a PersistentVolumeClaim remained in `Pending` state indefinitely. Running `kubectl get pods` showed:

```
NAME                       READY   STATUS    RESTARTS   AGE
web-app-0                  0/1     Pending   0          12m
web-app-1                  0/1     Pending   0          12m
db-statefulset-0           0/1     Pending   0          15m
cache-0                    0/1     Pending   0          8m
```

2. **kubectl describe pod** revealed the volume errors:

```
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Warning  FailedAttachVolume  45s   attachdetach-controller  Failed to attach volume "vol-xxxxxxxxxxxxx": rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Warning  FailedMount         30s   kubelet                  Unable to attach or mount volumes: unmounted volumes=[data], unattached volumes=[data default-token-xxxx]: timed out waiting for the condition
```

3. **StatefulSet Rolling Update Failure**: A manual rolling update of a StatefulSet caused it to hang — the new pod would get stuck, and the old pod would remain `Terminating`:

```
NAME                       READY   STATUS        RESTARTS   AGE
web-app-0                  0/1     Pending       0          2m
web-app-1                  1/1     Running       0          48m
web-app-2                  0/1     Terminating   0          48m
```

4. **PVC Status — Pending**: The PVC itself showed a `Pending` status with no volume attachment:

```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-web-app-0 Pending                                      gp3            12m
```

5. **Volume Already Used Error**: Some pods showed a different but equally confusing error:

```
Warning  FailedAttachVolume  20s   attachdetach-controller  Volume is already used by pod "web-app-0" that is not running
```

---

## Background

### Kubernetes Storage Architecture

Kubernetes storage is managed through three fundamental objects:

- **PersistentVolume (PV)**: A piece of storage provisioned in the cluster, with its own lifecycle independent of any pod. PVs are cluster resources.
- **PersistentVolumeClaim (PVC)**: A request for storage by a user. PVCs consume PV resources. Claims can request specific size, access modes, and storage classes.
- **StorageClass**: Describes the "classes" of storage. Different classes map to different quality-of-service levels, backup policies, or to custom provisions — determined by the provisioner field.

The flow is: User creates a PVC → Kubernetes finds a matching PV (or dynamically provisions one via the StorageClass provisioner) → The PVC is bound to the PV → Pods reference the PVC → The volume is attached and mounted.

### CSI Driver Model

The Container Storage Interface (CSI) is a standard for exposing storage systems to container workloads. A CSI driver consists of two main components:

1. **Controller Plugin** (StatefulSet): Runs as a Deployment in the cluster. Handles volume provisioning, attach/detach, and snapshot operations. Communicates with the cloud provider API (e.g., AWS EC2 API for EBS volume attachment).

2. **Node Plugin** (DaemonSet): Runs on every node. Handles volume mounting, formatting, and mounting into pods. Consists of several containers:
   - `csi-driver`: The actual storage driver that interacts with the storage backend
   - `node-driver-registrar`: A sidecar that registers the CSI driver with kubelet via the kubelet plugin registration mechanism
   - `liveness-probe`: Health checking for the CSI driver

The lifecycle of a CSI volume involves:
1. **CreateVolume**: CSI controller provisions the storage volume (e.g., creates an EBS volume)
2. **ControllerPublishVolume**: Attaches the volume to the node (e.g., EC2 AttachVolume)
3. **NodeStageVolume**: Formats and mounts the volume on the node
4. **NodePublishVolume**: Bind-mounts the volume into the pod's filesystem
5. Reverse operations for detach and delete

### Volume Attach/Detach Lifecycle

The attach/detach controller in `kube-controller-manager` is responsible for managing volume attachment operations. When a pod is scheduled to a node:

1. The scheduler informs the controller that a volume needs to be attached to a specific node
2. The controller calls `ControllerPublishVolume` on the CSI driver
3. The CSI controller plugin calls the cloud provider API to physically attach the volume
4. Once attached, the kubelet on the node discovers the device and proceeds with mounting

If any step in this chain fails, the pod remains stuck in `Pending`.

### fsGroup and Pod Security Context

In Kubernetes, `securityContext.fsGroup` specifies a supplemental group that applies to all containers in a pod. When a volume is mounted, kubelet recursively changes the ownership and permissions of the volume's contents to match this group. This is necessary for stateful applications that need write access to mounted volumes.

In Kubernetes 1.28 and later, the behavior of `fsGroup` changed significantly. Previously, kubelet would always perform recursive permission changes. Starting from 1.28, the CSI driver must explicitly declare its `fsGroupPolicy` in the `CSIDriver` object:

- `None` (default): The CSI driver does not support any fsGroup policy. Kubelet will NOT perform permission changes on mount.
- `File`: The CSI driver supports fsGroup via file-level permission changes. Kubelet will perform recursive `chown`/`chmod` on the volume.
- `ReadWriteOnceWithFSType`: The CSI driver supports fsGroup only for volumes that are not shared (access mode ReadWriteOnce) and for specific filesystem types.

If `fsGroupPolicy` is not set or is set to `None`, stateful pods that rely on `fsGroup` for filesystem permissions will fail because kubelet skips the permission fix.

---

## Investigation

### Step 1: Check Pod Status

The first sign of trouble was the high number of pending pods. We ran:

```bash
kubectl get pods -A | grep Pending
```

This revealed more than 40 pods stuck in `Pending` state across multiple namespaces. All of them were pods with PVC mounts.

We inspected one of the stuck pods:

```bash
kubectl describe pod web-app-0
```

The Events section showed:

```
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Warning  FailedAttachVolume  2m    attachdetach-controller  Failed to attach volume "vol-0a1b2c3d4e5f": rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

**Key observation**: The attach operation was timing out. The controller was able to initiate the attach but the CSI driver was not responding within the deadline.

### Step 2: Check PVC/PV Status

```bash
kubectl get pvc -A
```

Output showed all newly created PVCs in `Pending` status:

```
NAMESPACE   NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
default     data-web-app-0    Pending                                      gp3            15m
default     data-web-app-1    Pending                                      gp3            12m
database    data-db-0         Pending                                      gp3            10m
```

Looking at a specific PVC:

```bash
kubectl describe pvc data-web-app-0
```

The describe output showed that the PVC was waiting for a volume to be created — the StorageClass provisioner had not created the EBS volume yet.

### Step 3: Check CSI Driver Pods

We checked the CSI driver components:

```bash
kubectl get pods -n kube-system -l app=ebs-csi-node
```

The node plugin DaemonSet pods were all running:

```
NAME                    READY   STATUS    RESTARTS   AGE
ebs-csi-node-abc12      3/3     Running   0          6h
ebs-csi-node-def34      3/3     Running   0          6h
ebs-csi-node-ghi56      2/3     Running   0          6h
ebs-csi-node-jkl78      3/3     Running   0          6h
```

Wait — one node showed `2/3` instead of `3/3`. That was suspicious. One of the three containers in the CSI node plugin was not ready.

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

The controller plugin was also running:

```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-7d8f9c6b8c-ab12   2/2     Running   0          6h
ebs-csi-controller-7d8f9c6b8c-cd34   2/2     Running   0          6h
```

### Step 4: Check CSI Driver Logs

We checked the logs of the node plugin container that had `2/3` readiness:

```bash
kubectl logs -n kube-system ebs-csi-node-ghi56 csi-driver
```

The log showed repeated registration errors:

```
I0610 10:15:23.456789       1 main.go:114] Version: v1.20.0
I0610 10:15:23.456912       1 main.go:127] Driver: ebs.csi.aws.com
E0610 10:15:23.457123       1 server.go:82] Failed to get driver capabilities: node service capability not supported
E0610 10:15:23.457156       1 main.go:135] Failed to start driver: node service capability not supported
```

This was a critical finding — the CSI driver version was too old and did not support the node service capabilities required by K8s 1.29.

We also checked the controller plugin logs:

```bash
kubectl logs -n kube-system ebs-csi-controller-7d8f9c6b8c-ab12 csi-provisioner
```

The provisioner logs showed:

```
W0610 10:15:30.123456       1 connection.go:182] Still connecting to unix:///csi/csi.sock, error: connection error
I0610 10:15:30.789012       1 controller.go:144] CreateVolume: called with args: {name: pvc-xxxxxxxxxx}
E0610 10:15:30.789156       1 controller.go:155] Failed to create volume: rpc error: code = Unavailable desc = all SubConns are in TransientFailure
```

The CSI controller was unable to communicate with the storage backend, which explained why volumes were not being provisioned.

### Step 5: Check CSIDriver Object

```bash
kubectl get csidriver ebs.csi.aws.com -o yaml
```

This was a key moment in the investigation:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
```

Notice what was **missing**: there was NO `fsGroupPolicy` field. In K8s 1.29, when `fsGroupPolicy` is not specified, it defaults to `None`. This meant kubelet would never perform permission fixes on mounted volumes, causing pods with `securityContext.fsGroup` to fail.

### Step 6: Check Node CSI Registration

```bash
ls -la /var/lib/kubelet/plugins/ebs.csi.aws.com/
```

On some nodes the CSI plugin socket file was missing:

```
total 0
```

On nodes where the socket was present, we checked the registration status:

```bash
ls -la /var/lib/kubelet/plugins_registry/
```

The node-driver-registrar was not registering the driver properly with kubelet, as confirmed by the logs showing "node service capability not supported".

### Step 7: Check Kubelet Logs

```bash
journalctl -u kubelet | grep -i "volume\|pvc\|csi"
```

The kubelet logs confirmed the issue:

```
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:15.123456    1234 reconciler.go:256] volume_attachment "vol-0a1b2c3d4e5f" has error: "timeout waiting for attachment to complete"
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:15.789012    1234 kubelet_volumes.go:154] Could not mount volume "pvc-xxxxxxxxxx" (vol-0a1b2c3d4e5f): context deadline exceeded
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: W0610 10:20:15.789123    1234 volume_manager.go:487] Volume "pvc-xxxxxxxxxx" is not attached to node "ip-10-0-1-23"
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:16.123789    1234 plugin_manager.go:136] Registration of plugin "ebs.csi.aws.com" failed: node-driver-registrar reported error: driver responded with an error
```

The registration failure and volume attach timeout were clearly correlated.

### Step 8: Check StorageClass

```bash
kubectl get sc gp3 -o yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

The StorageClass looked correct — it pointed to the right CSI provisioner, used `WaitForFirstConsumer` (which meant the volume would only be provisioned after a pod was scheduled), and had the correct filesystem type. This confirmed that the StorageClass itself was not the problem.

---

## Root Cause

The investigation revealed three interconnected issues, all stemming from the same root cause: **the CSI driver version was incompatible with the Kubernetes 1.29 upgrade**.

### Issue 1: CSI Driver Version Mismatch

The cluster was running EBS CSI driver `v1.20.0`, which was released before K8s 1.29 was available. This version did not implement the CSI node service capabilities that K8s 1.29's kubelet required. Specifically:

- The `node-driver-registrar` sidecar (included in the CSI driver) was too old to register with the updated kubelet plugin registration protocol
- The CSI driver's `NodeGetInfo` RPC did not return capabilities that K8s 1.29 expected
- The driver's `NodeStageVolume` and `NodePublishVolume` calls failed because the driver was not properly registered

### Issue 2: Missing fsGroupPolicy in CSIDriver Spec

Kubernetes 1.28 introduced the `fsGroupPolicy` field for CSIDriver objects. When a CSIDriver is created or updated without this field:

- In K8s 1.27 and earlier: kubelet handled fsGroup changes by default
- In K8s 1.28+: the field defaults to `None`, meaning kubelet **skips** all fsGroup permission fixes

The EBS CSI driver v1.20.0 did not set `fsGroupPolicy` because this field did not exist when the driver was developed. After the K8s upgrade, the CSIDriver object continued to work without the field, but kubelet now interpreted the missing field as `fsGroupPolicy: None`, which caused:

- Stateful pods that relied on `securityContext.fsGroup` for filesystem write access could not start
- Volume mount permission checks failed silently
- Pods remained in `Pending` state because the mount could not complete

### Issue 3: VolumeAttach Timeout Cascade

Because the node plugin was not properly registered, the attach/detach controller could complete the `ControllerPublishVolume` call (which makes the EC2 API call to attach the volume to the instance), but the node-side operations (discovering the device, staging, and publishing the mount) would fail. This created a cascade:

1. The PVC requested a volume
2. The CSI controller created the EBS volume successfully
3. The attach/detach controller called `ControllerPublishVolume` and the EBS volume was attached to the EC2 instance
4. But the kubelet on the node could not discover or mount the device because the CSI driver was not registered
5. The attach timeout expired, Kubernetes retried, but the registration failure persisted
6. Pods remained stuck in `Pending`

### The "Volume Already Used" Error

Some pods showed `"Volume is already used by pod that is not running"`. This occurred because:

- During a StatefulSet rolling update, the old pod is terminated but its PVC remains bound
- The new pod is created and tries to attach the same PVC
- The attach/detach controller sees that the volume is still "attached" to the old pod (which has been deleted but the detach operation never completed because the CSI driver was malfunctioning)
- The controller refuses to attach the volume to the new pod, producing this misleading error message

---

## Resolution

### Emergency Fix — Upgrade CSI Driver

The immediate fix was to upgrade the EBS CSI driver to a version compatible with Kubernetes 1.29.

```bash
# Deploy the stable overlay with a compatible version
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.30.0"
```

This command:

1. Deployed the CSI driver controller (StatefulSet) with the updated image
2. Deployed the CSI driver node plugin (DaemonSet) with updated images across all nodes
3. Updated the CSIDriver object with the new spec (including `fsGroupPolicy: File`)
4. Applied necessary RBAC and ServiceAccount updates

After the upgrade, we verified the CSIDriver object:

```bash
kubectl get csidriver ebs.csi.aws.com -o yaml
```

The updated output now showed:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true
  fsGroupPolicy: File
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
```

The critical change was `fsGroupPolicy: File` — this told kubelet to perform the recursive permission changes needed by stateful pods.

### Clean Up Stuck Pods

After upgrading the CSI driver, we needed to clear the backlog of stuck pods:

```bash
# Delete all pending pods — they will be recreated by their controllers
kubectl delete pods --field-selector=status.phase=Pending -A
```

This triggered the StatefulSet and Deployment controllers to recreate the pods. With the updated CSI driver, the new pods were able to attach and mount volumes successfully.

### Immediate Patch (Alternative)

If the full CSI driver upgrade could not be performed immediately (e.g., approval needed), a quick workaround was to patch the CSIDriver object manually:

```bash
kubectl patch csidriver ebs.csi.aws.com -p '{"spec":{"fsGroupPolicy":"File"}}'
```

This would fix the fsGroup issue immediately but would NOT fix the node-driver-registrar compatibility issue. The node plugin still needed to be upgraded to register properly with kubelet.

### Verify Recovery

We verified that the fix was working:

```bash
# Check all pods are running
kubectl get pods -A | grep -v Running | grep -v Completed

# Check PVCs are bound
kubectl get pvc -A | grep -v Bound

# Check CSI node pods are all 3/3
kubectl get pods -n kube-system -l app=ebs-csi-node

# Perform a test rolling update
kubectl rollout restart statefulset web-app
```

All pods transitioned to `Running`, PVCs showed `Bound` status, and rolling updates completed without issues.

### Long-Term Prevention

To prevent this from happening again, we implemented the following measures:

**1. Upgrade Checklist Additions**

Added the following checks to the cluster upgrade runbook:

```
Pre-Upgrade Checks:
  - Verify CSI driver version compatibility with target K8s version
  - Run: kubectl get csidriver -o yaml
  - Verify fsGroupPolicy is set correctly (File or ReadWriteOnceWithFSType)
  - Verify node-driver-registrar image version in CSI DaemonSet
  - Check Kubernetes changelog for storage-related breaking changes (e.g., fsGroupPolicy, in-tree storage plugin removals)

Post-Upgrade Validation:
  - Create a test PVC and pod, verify mount succeeds
  - Verify CSI driver registration: ls /var/lib/kubelet/plugins/<driver-name>/
  - Perform a StatefulSet rolling update end-to-end
  - Monitor volume_attach_status and csi_volume_op_total metrics
```

**2. Monitoring and Alerts**

Added Prometheus recording rules and alerts:

```
# Alert: CSI Driver Registration Failure
- alert: CSIDriverRegistrationFailure
  expr: rate(csi_plugin_registrations_total{status="failure"}[5m]) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "CSI driver registration failures detected"

# Alert: Volume Attach Timeout
- alert: VolumeAttachTimeout
  expr: rate(volume_operation_total_seconds_sum{status="fail"}[5m]) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Volume attach operations are failing"

# Record: CSI Driver Version
- record: csi_driver_version
  expr: csv{driver="ebs.csi.aws.com"}
```

**3. CI/CD Pipeline Checks**

Added version compatibility validation to the deployment pipeline:

```bash
# Check CSI driver compatibility before K8s upgrade
verify_csi_compatibility() {
  local current_k8s_version="$1"
  local target_k8s_version="$2"
  local csi_driver_image="$3"

  # Retrieve the CSI driver's supported K8s versions from the image metadata
  # Fail the pipeline if the CSI driver does not declare support for the target version
  echo "Verifying CSI driver $csi_driver_image compatibility with K8s $target_k8s_version..."
}
```

**4. Volume Lifecycle Testing**

Added a post-upgrade test suite that validates the full volume lifecycle:

- Provision a test volume
- Attach the volume to a node
- Mount the volume into a pod
- Write data and verify fsGroup permissions
- Unmount and detach
- Delete the volume

This test is run automatically after every K8s upgrade in a non-production environment.

---

## Lessons Learned

### What Went Wrong

1. **Lack of pre-upgrade compatibility validation**: The CSI driver version was not checked against the target Kubernetes version before the upgrade. A simple check like `kubectl get csidriver` and reviewing the CSI driver release notes could have caught the incompatibility.

2. **Missing fsGroupPolicy awareness**: The team was not aware that K8s 1.28 had introduced a breaking change requiring CSI drivers to explicitly declare `fsGroupPolicy`. This was documented in the Kubernetes changelog but not incorporated into the upgrade checklist.

3. **Insufficient post-upgrade validation**: The post-upgrade checks only verified that control plane components were healthy (kube-apiserver, controller-manager, scheduler). They did not include volume lifecycle testing for stateful workloads.

4. **Assumption that CSI upgrades follow K8s upgrades**: The CSI driver was treated as a "stable infrastructure component" that did not need to be updated alongside the Kubernetes control plane. In reality, CSI driver versions are tightly coupled to Kubernetes versions.

### What We Did Well

1. **Systematic investigation**: The investigation followed a logical progression: pod status → PVC status → CSI driver health → CSIDriver object → components logs, which efficiently identified the root cause.

2. **Immediate mitigation**: Patching the CSIDriver object with the correct `fsGroupPolicy` and upgrading the CSI driver resolved the issue quickly once identified.

3. **Monitoring integration**: Existing monitoring alerted the team within minutes of the node group refresh, minimizing the time to detection.

### Key Takeaways

- **Storage and CSI driver compatibility must be validated before every K8s upgrade** — not just control plane and node compatibility.
- **The CSIDriver object spec changes over time** — always review `kubectl get csidriver -o yaml` against the Kubernetes version you are upgrading to.
- **Stateful workloads require volume lifecycle testing as part of upgrade validation** — verifying that control plane components are healthy is not sufficient.
- **CSI driver versions are tied to Kubernetes versions** — always check the CSI driver's release notes for supported Kubernetes versions before upgrading.

---

## Summary

### Incident Timeline

| Time | Event |
|------|-------|
| T-0 | K8s control plane upgrade from v1.28 to v1.29 completed successfully |
| T+5m | Node group refresh initiated — new nodes launched with v1.29 kubelet |
| T+10m | Alerts triggered: stateful pods stuck in Pending |
| T+15m | On-call engineer begins investigation |
| T+30m | CSI driver logs show registration failures in node plugin |
| T+45m | CSIDriver object found without fsGroupPolicy field |
| T+60m | RCA identified: CSI driver v1.20 incompatible with K8s 1.29 |
| T+75m | CSI driver upgraded to v1.30.0 |
| T+80m | CSIDriver fsGroupPolicy verified as File |
| T+85m | Stuck pods deleted — controllers recreate them successfully |
| T+100m | All stateful workloads recovered |

### Version Compatibility

| Kubernetes Version | Required CSI Driver Version | Required node-driver-registrar | fsGroupPolicy |
|--------------------|---------------------------|-------------------------------|---------------|
| v1.27 and earlier | v1.20 (worked) | v2.5.x | Not required |
| v1.28 | v1.25+ | v2.7+ | Must be set |
| v1.29 | v1.30+ | v2.9+ | Must be set to File |
| v1.30 | v1.30+ | v2.9+ | Must be set to File |

### Key Commands Reference

```bash
# Check pod volume errors
kubectl describe pod <pod-name>

# Check PVC status
kubectl get pvc -A
kubectl describe pvc <pvc-name>

# Check CSIDriver object
kubectl get csidriver -o yaml

# Check CSI driver node plugin logs
kubectl logs -n kube-system -l app=ebs-csi-node csi-driver

# Check CSI driver controller logs
kubectl logs -n kube-system -l app=ebs-csi-controller csi-provisioner

# Upgrade CSI driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.30.0"

# Patch CSIDriver fsGroupPolicy (emergency)
kubectl patch csidriver ebs.csi.aws.com -p '{"spec":{"fsGroupPolicy":"File"}}'

# Delete all pending pods to force recreation
kubectl delete pods --field-selector=status.phase=Pending -A

# Check kubelet logs for volume errors
journalctl -u kubelet | grep -i "volume\|pvc\|csi"

# Check node CSI plugin socket
ls -la /var/lib/kubelet/plugins/ebs.csi.aws.com/
```

This incident served as a valuable reminder that Kubernetes storage is a complex subsystem with multiple moving parts, and that version compatibility across these components must be carefully managed during cluster upgrades. The most deceptive aspect was that existing pods continued to work — only new pod creation and rolling updates were affected — which made the root cause less obvious during initial troubleshooting. Systematic investigation following the storage stack hierarchy (pod → PVC → CSI driver → CSIDriver spec → kubelet registration) was essential to identifying and resolving the issue.
