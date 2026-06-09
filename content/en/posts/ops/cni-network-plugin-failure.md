---
title: "CNI Network Plugin Failure — When Pods Couldn't Talk to Each Other Anymore"
date: 2026-06-10
weight: 100470
slug: "cni-network-plugin-failure"
tags: ["kubernetes", "networking", "cni", "calico", "troubleshooting"]
categories: ["Troubleshooting"]
description: "A CNI network plugin failure incident — how a Calico configuration mismatch after a cluster upgrade caused pods to lose network connectivity, breaking inter-service communication across the entire cluster"
keywords: "cni network plugin failure, calico troubleshooting, kubernetes pod network broken, calico felix configuration, kubernetes cni not working"
draft: false
featured: true
cover:
  image: ""
  caption: "CNI Network Plugin Failure — Troubleshooting"
---

# CNI Network Plugin Failure — When Pods Couldn't Talk to Each Other Anymore

## Common Search Queries

If you're landing here from a search, this post covers:

- cni network plugin failure kubernetes pod containercreating
- network is not ready calico fix
- failed to create pod sandbox error adding pod to cni
- calico felix ipip tunnel not working after upgrade
- calico vxlan vs ipip migration troubleshooting
- kubernetes cni not working after control plane upgrade
- calicoctl node status shows no connectivity
- kubectl describe pod network not ready cni

---

## The Incident

### Environment

| Component | Version / Detail |
|-----------|-----------------|
| Kubernetes | 1.29 (upgraded from 1.28) |
| CNI Plugin | Calico v3.26 |
| Cluster Size | 30 worker nodes |
| Pod Count | ~500 pods |
| Overlay Mode | IPIP (IP-in-IP tunneling) |
| Node OS | Ubuntu 22.04 LTS |

### Timeline

**09:30 UTC** — The platform team started a routine Kubernetes control plane upgrade from v1.28 to v1.29 across a production cluster. The upgrade procedure followed the standard `kubeadm upgrade` flow: first the control plane nodes, then the worker nodes.

**10:15 UTC** — Control plane upgrade completed. Worker node upgrades began. The team noticed that newly created pods across the cluster were stuck in `ContainerCreating` state.

**10:20 UTC** — `kubectl describe pod` on affected pods returned `network is not ready` and `failed to create pod sandbox: error adding pod to CNI`. Existing pods continued to function normally, but any new deployment or rolling update failed.

**10:25 UTC** — The team declared a P0 incident. All new deployments, horizontal pod autoscaler triggers, and rolling updates were effectively blocked.

### Symptoms

- **New pods stuck in `ContainerCreating`**: Every pod created after the upgrade window remained in `ContainerCreating` indefinitely.
- **Existing pods worked**: Pods that were running before the upgrade had normal network connectivity and could communicate with each other.
- **"network is not ready" errors**: The kubelet reported the node network was not ready for new pods.
- **CNI plugin errors**: `kubectl describe pod` showed:
  ```
  Warning  FailedCreatePodSandBox  2m   kubelet  Failed to create pod sandbox: rpc error:
  code = Unknown desc = failed to setup network for sandbox "...":
  plugin type="calico" failed (add): error adding pod to CNI network
  ```
- **Cluster-wide impact**: Although existing services stayed up, any operation requiring new pod creation — deployments, StatefulSets, DaemonSets, HPA scale-ups, and CI/CD pipelines — was completely blocked.

---

## Background

To understand what went wrong, we need a quick refresher on how the CNI plugin architecture works in Kubernetes.

### CNI Plugin Architecture

When the kubelet on a node needs to create a pod, the workflow is:

```
kubelet → CRI (containerd/CRI-O) → Create sandbox → CNI plugin → Network setup
```

1. The kubelet instructs the Container Runtime Interface (CRI) to create a pod sandbox.
2. The CRI runtime invokes the CNI plugin (in our case, Calico) to configure the pod's network namespace.
3. The CNI plugin sets up the virtual Ethernet pair (veth), assigns an IP address, and configures routing rules.
4. If an overlay network is used (IPIP or VXLAN), the CNI plugin also sets up encapsulation tunnels.

If any step in this chain fails, the pod sandbox creation fails and the pod stays in `ContainerCreating`.

### Calico Components

Calico consists of several key components:

| Component | Role |
|-----------|------|
| **Felix** | The core Calico agent running on each node. Manages network interfaces, routes, and ACL rules. Programs the Linux networking stack. |
| **BIRD** | The BGP route reflector client. Distributes routing information between nodes. |
| **confd** | Dynamically generates BIRD configuration from Calico datastore data. |
| **calicoctl** | Command-line tool for managing Calico configuration and troubleshooting. |

### IPIP vs VXLAN Overlay

Both IPIP and VXLAN are overlay networking technologies used to encapsulate pod traffic across the cluster's underlying network fabric:

| Feature | IPIP | VXLAN |
|---------|------|-------|
| Encapsulation | IP-in-IP (Protocol 4) | MAC-in-UDP |
| Overhead | 20 bytes per packet | 50 bytes per packet |
| Kernel Module | `ipip.ko` | `vxlan.ko` |
| Use Case | Simple L3 networks | L2 networks, cloud environments |
| MTU Overhead | 1440 (on 1500 MTU fabric) | 1450 (on 1500 MTU fabric) |

The key difference that matters for our incident: **IPIP requires the `ipip` kernel module to be loaded**, while VXLAN requires the `vxlan` module. If these modules are not available, the tunnel interface cannot be created and the CNI plugin will fail.

---

## Investigation

The investigation followed a systematic narrowing-down process. Here is every step we took, in order.

### Step 1: Check Pod Status

```bash
kubectl get pods -A | grep ContainerCreating
```

This showed dozens of pods across multiple namespaces stuck in `ContainerCreating`. Existing pods were all `Running`.

```bash
kubectl describe pod <stuck-pod> -n <namespace>
```

The output confirmed:

```
Events:
  Type     Reason                  Age   From               Message
  ----     ------                  ----  ----               -------
  Warning  FailedCreatePodSandBox  10s   kubelet            Failed to create pod sandbox:
    rpc error: code = Unknown desc = failed to setup network for sandbox
    "abc123...": plugin type="calico" failed (add): error adding pod to CNI network
```

**Takeaway**: The error was clearly coming from the CNI plugin layer. Kubelet was attempting to set up the pod network, but Calico was refusing or unable to complete the operation.

### Step 2: Check Kubelet Logs

```bash
journalctl -u kubelet --since "30 min ago" | grep -i cni
```

This revealed repeated errors:

```
kubelet[1234]: E0610 10:16:22.123456    1234 cni.go:320] Error validating CNI config:
  error loading config file /etc/cni/net.d/10-calico.conflist: error parsing config:
  unknown plugin type "calico"
```

Wait — this error is misleading. Calico was definitely a valid plugin. The real issue was deeper. Further kubelet logs showed:

```
kubelet[1234]: E0610 10:17:01.654321    1234 kubelet.go:234] "Pod sandbox setup failed"
  pod="namespace/pod-name" error="failed to setup network for sandbox "...":
  plugin type="calico" failed (add): error adding pod to CNI network
```

The kubelet was correctly finding the Calico CNI config but Calico was failing during the `ADD` operation.

### Step 3: Check Calico Pods

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

```
NAME                READY   STATUS    RESTARTS   AGE
calico-node-abc12   1/1     Running   0          12h
calico-node-def34   1/1     Running   0          12h
calico-node-ghi56   1/1     Running   0          12h
...
```

All Calico node pods were `Running`. This ruled out a crash or deployment failure in Calico itself.

### Step 4: Check Calico Node Logs

```bash
kubectl logs -n kube-system calico-node-abc12
```

The logs showed a repeating pattern:

```
2026-06-10 10:15:30.123 [INFO][1234] felix/int_dataplane.go  : Starting dataplane driver: linux
2026-06-10 10:15:30.456 [INFO][1234] felix/int_dataplane.go  : Linux dataplane driver started
2026-06-10 10:15:30.789 [WARN][1234] felix/ipip_mgr.go       : IPIP tunnel setup failed:
  Failed to create IPIP tunnel tunl0: error creating tunnel: operation not supported
2026-06-10 10:15:31.012 [WARN][1234] felix/ipip_mgr.go       : IPIP tunnel setup failed:
  Failed to create IPIP tunnel tunl0: error creating tunnel: operation not supported
```

This was the critical clue. **Felix was repeatedly failing to create the IPIP tunnel**. The error `operation not supported` pointed directly to a missing kernel module or a kernel that doesn't support IPIP encapsulation.

### Step 5: Check CNI Config on Node

```bash
cat /etc/cni/net.d/10-calico.conflist
```

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "worker-node-01",
      "mtu": 1440,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
```

The CNI config looked normal. The MTU was set to 1440, which is the standard value for IPIP encapsulation on a 1500 MTU fabric.

### Step 6: Check Calico Felix Configuration

```bash
kubectl exec -n kube-system calico-node-abc12 -- cat /etc/calico/calico.cfg
```

No output? That's because modern Calico (v3.26+) reads configuration from the Kubernetes API via `FelixConfiguration` CRDs. Let's check those:

```bash
kubectl get FelixConfiguration default -o yaml
```

```yaml
apiVersion: crd.projectcalico.org/v1
kind: FelixConfiguration
metadata:
  name: default
spec:
  ipipEnabled: true
  ipip:
    enabled: true
    mode: Always
  vxlanEnabled: false
  mtu: 1440
```

This confirmed: IPIP was enabled and set to `Always` mode, while VXLAN was disabled.

### Step 7: Check IP Tunnels on Node

```bash
ip tunnel show
```

If IPIP were working, this would show:

```
tunl0: ip/ip  remote any  local any  ttl inherit  nopmtudisc
```

But instead, the output was **empty** or showed an error. The `tunl0` interface simply did not exist on the upgraded nodes.

```bash
ip link show tunl0
```

```
Device "tunl0" does not exist.
```

This confirmed that the IPIP tunnel device was never created.

### Step 8: Check Kernel Modules

```bash
lsmod | grep ipip
```

No output — the `ipip` module was not loaded.

```bash
lsmod | grep tunnel
```

This showed `tunnel4` and `tunnel6` were present, but `ipip` was missing.

```bash
modprobe ipip
```

This command completed without error. After loading the module:

```bash
lsmod | grep ipip
```

```
ipip                   16384  0
tunnel4                16384  1 ipip
```

The module was now loaded. But this only fixed the current node — a reboot would lose it.

### Step 9: Test Node-to-Node Connectivity with calicoctl

```bash
kubectl exec -n kube-system calico-node-abc12 -- calicoctl node status
```

```
Calico process is running.

IPv4 BGP status
No IPv4 peers found.
```

The BGP status showed no peers — Calico had no established BGP sessions with neighboring nodes. This was a consequence of the IPIP tunnel failure: without working tunnels, the BGP routes could not be exchanged properly.

---

## Root Cause

The root cause was a **kernel module availability mismatch** introduced by the Kubernetes upgrade from v1.28 to v1.29.

### What Happened

1. **The upgrade process updated node images**: When the worker nodes were upgraded via `kubeadm upgrade`, the process involved updating system packages and, in some configurations, provisioning new node images. The upgraded nodes shipped with a kernel where the `ipip.ko` module was present on disk but **not loaded by default**.

2. **Calico required IPIP**: The Calico `FelixConfiguration` specified `ipipEnabled: true` with `IPIPMode: Always`. This meant Calico Felix would attempt to create an IPIP tunnel (`tunl0`) on every node during startup.

3. **IPIP tunnel creation failed**: When Felix tried to create the `tunl0` interface, the Linux kernel returned `operation not supported` because the `ipip` kernel module was not loaded. Without this module, the kernel doesn't recognize IPIP encapsulation (protocol 4).

4. **Calico Felix entered a degraded state**: Felix continued running and reporting errors, but could not complete the network setup for new pods. Existing pod network namespaces were untouched — their veth pairs and routes remained intact — which is why existing pods still worked.

5. **The CNI ADD operation failed**: When the kubelet asked Calico to add a new pod to the network, the CNI plugin attempted to create veth pairs and add routes that depended on the IPIP tunnel infrastructure. Since the tunnel wasn't available, the entire operation failed, and the kubelet left the pod in `ContainerCreating`.

### Why Existing Pods Still Worked

This is a frequently misunderstood aspect of CNI failures. Existing pods continued to function because:

- Their network namespaces were already created and configured before the upgrade
- The veth pairs connecting their namespaces to the host were still intact
- The Linux routing table entries for existing pods were still valid
- The IPIP tunnel, while missing, only affects **new** network namespace creation

Think of it like a bridge that collapsed after cars already crossed — the cars on the far side can still drive, but no new cars can cross.

### The MTU Angle

Calico was configured with `mtu: 1440`, which is correct for IPIP on a 1500 MTU fabric (1500 - 20 byte IPIP header = 1480, but Calico also accounts for potential Kubernetes service encapsulation, resulting in 1440). However, after the upgrade, since IPIP was non-functional, this MTU setting became irrelevant for new connections.

---

## Resolution

We had two paths to resolution: an immediate emergency fix (load the IPIP module) and a permanent configuration fix (switch to VXLAN).

### Emergency Fix: Load IPIP Kernel Module

The quickest way to restore service was to load the missing kernel module on every node:

```bash
# On every worker node
modprobe ipip

# Verify it loaded
lsmod | grep ipip
```

Then restart Calico to pick up the change:

```bash
kubectl rollout restart -n kube-system daemonset calico-node
```

Wait for the Calico pods to restart and verify:

```bash
kubectl rollout status -n kube-system daemonset calico-node --timeout=5m

# Verify IPIP tunnel is created
kubectl exec -n kube-system calico-node-abc12 -- ip tunnel show
# Expected: tunl0: ip/ip  remote any  local any  ttl inherit

# Verify BGP peers
kubectl exec -n kube-system calico-node-abc12 -- calicoctl node status
```

Finally, delete all stuck pods to allow them to be recreated:

```bash
kubectl delete pods --field-selector=status.phase=Pending -A
```

**Important**: This fix is temporary. A node reboot would unload the `ipip` module, and the problem would recur. To make it permanent, either:
- Add `ipip` to `/etc/modules-load.d/` so it loads on boot, or
- Switch to VXLAN mode (recommended)

### Permanent Fix: Switch to VXLAN Mode

Given that VXLAN is more portable across kernel versions and cloud environments, and avoids the IPIP kernel module dependency issue, we chose to migrate the Calico overlay to VXLAN.

**Step 1: Update the Calico Installation resource**

```bash
kubectl edit installation.operator.tigera.io default
```

Change the encapsulation mode from IPIP to VXLAN:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet  # Was: IPIPCrossSubnet
    mtu: 1450  # VXLAN overhead is 50 bytes
```

**Step 2: Apply the change**

```bash
kubectl apply -f calico-installation.yaml
```

**Step 3: Restart Calico**

```bash
kubectl rollout restart -n kube-system daemonset calico-node
```

**Step 4: Restart all existing workloads**

This is necessary because existing pods still have IPIP-based routes and veth pairs. A full pod restart is needed to switch them to VXLAN:

```bash
# Restart all deployments (be careful with prod traffic!)
kubectl get deployments -A -o json | jq -r '.items[] | "kubectl rollout restart deployment -n \(.metadata.namespace) \(.metadata.name)"' | bash
```

**Step 5: Verify the switch**

```bash
# Check that VXLAN interfaces exist
kubectl exec -n kube-system calico-node-abc12 -- ip link show

# Look for interfaces like vxlan.calico

# Verify BGP status
calicoctl node status
```

### Long-term Preventive Measures

After resolving the incident, we implemented the following safeguards:

#### 1. CNI Pre-flight Check Script

Add a pre-flight check to the cluster upgrade procedure that validates the CNI plugin before draining nodes:

```bash
#!/bin/bash
# cni-preflight-check.sh

echo "=== CNI Pre-flight Check ==="

# Check kernel modules
for mod in ipip vxlan; do
  if lsmod | grep -q "^$mod"; then
    echo "[OK] Kernel module $mod is loaded"
  else
    echo "[WARN] Kernel module $mod is NOT loaded"
    modprobe $mod 2>/dev/null && echo "  -> Successfully loaded $mod" || echo "  -> FAILED to load $mod"
  fi
done

# Check Calico node status
kubectl exec -n kube-system daemonset/calico-node -- calicoctl node status 2>/dev/null || echo "[FAIL] calicoctl node status failed"

# Check tunnel interfaces
kubectl exec -n kube-system daemonset/calico-node -- ip tunnel show 2>/dev/null

# Check Calico Felix health
kubectl get felixconfiguration default -o yaml | grep -E "ipipEnabled|vxlanEnabled"

echo "=== Check Complete ==="
```

#### 2. Calico Health as Node Readiness Probe

Add a custom node readiness check that verifies Calico functionality:

```yaml
# This can be added as a script or a custom controller
# to mark nodes as NotReady if Calico is degraded
```

#### 3. Monitoring Calico Felix Metrics

Add Prometheus alerting for Calico Felix resync state:

```yaml
# PrometheusRule example
groups:
- name: calico.rules
  rules:
  - alert: CalicoFelixResyncStalled
    expr: calico_node_felix_resync_state > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Calico Felix resync stalled on {{ $labels.node }}"
```

#### 4. Version Compatibility Matrix

Pin the Calico version to a K8s-compatible release and document the matrix:

| Kubernetes | Calico (Recommended) |
|-----------|---------------------|
| 1.28 | v3.26, v3.27 |
| 1.29 | v3.27, v3.28 |
| 1.30 | v3.28+ |

#### 5. Kernel Module Check in Node Bootstrap

Add kernel module validation to the node initialization / bootstrap script (whether using `kubeadm`, `Terraform`, or a custom provisioner):

```bash
# In node bootstrap script
REQUIRED_MODULES=("ipip" "vxlan" "tunnel4" "tunnel6" "br_netfilter" "overlay")
for mod in "${REQUIRED_MODULES[@]}"; do
  modprobe $mod || echo "FAILED to load $mod"
done

# Persist modules across reboots
cat > /etc/modules-load.d/calico.conf <<EOF
ipip
vxlan
tunnel4
tunnel6
br_netfilter
overlay
EOF
```

#### 6. Document IPIP vs VXLAN Tradeoffs

| Decision Factor | IPIP | VXLAN |
|----------------|------|-------|
| Performance | Slightly better (lower overhead) | Slightly worse (50 vs 20 bytes) |
| Portability | Requires `ipip.ko`; not available on all cloud VMs | More widely supported; works on most platforms |
| Cloud Compatibility | May not work on some cloud providers (e.g., GCP) | Works across all major cloud providers |
| MTU Overhead | 20 bytes | 50 bytes |
| Use Case | On-premise, controlled environments | Cloud, multi-tenant, hybrid environments |

---

## Lessons Learned

### What Went Well

- **Existing pods survived**: Because the upgrade didn't restart running pods, critical services remained available during the incident.
- **Calico error logging**: The Felix logs clearly showed `IPIP tunnel setup failed`, which pointed directly to the root cause once we looked in the right place.
- **Rollback capability**: The team had the `kubeadm upgrade revert` procedure documented, though it wasn't needed.

### What Went Wrong

- **No pre-flight check for CNI**: The upgrade procedure validated control plane components but did not validate that the CNI plugin would function correctly after the node image update.
- **Assumed kernel module availability**: We assumed that IPIP was built into the kernel or loaded by default on all node images. This was true for the old base image but not for the updated one.
- **No health monitoring for Calico**: We had node-level health checks (`NodeReady`) but no Calico-specific health monitoring. Felix's degraded state didn't trigger any alert.
- **MTU configuration not revisited**: When switching overlay modes, MTU needs to be adjusted. We had not validated the MTU configuration for VXLAN as a fallback.

### Key Takeaways

1. **CNI is a critical upgrade dependency**: Treat the CNI plugin as a first-class component during any cluster upgrade. Test it explicitly before upgrading worker nodes.
2. **Kernel modules are infrastructure**: Don't assume common kernel modules are loaded by default. Explicitly configure them in node bootstrap scripts.
3. **Monitor the network fabric**: Calico Felix metrics (especially `felix_resync_state` and `ipip_tunnel_errors`) should be part of your monitoring stack.
4. **Test pod creation in post-upgrade validation**: Don't just check that nodes are `Ready` — verify that new pods can actually be created and networked.
5. **Have a rollback plan for the CNI layer**: Know how to switch overlay modes quickly if your primary mode fails.

---

## Summary

### Timeline Recap

| Time | Event |
|------|-------|
| 09:30 UTC | Control plane upgrade from K8s 1.28 to 1.29 begins |
| 10:15 UTC | Worker nodes upgraded; new pods stuck in `ContainerCreating` |
| 10:20 UTC | Incident declared; investigation begins |
| 10:25 UTC | Calico Felix logs reveal IPIP tunnel creation failure |
| 10:35 UTC | `modprobe ipip` applied to first node; Calico restarted |
| 10:45 UTC | Emergency fix rolled out to all nodes |
| 11:00 UTC | Stuck pods deleted; new pods creating successfully |
| 11:30 UTC | Decision made to migrate to VXLAN permanently |
| 13:00 UTC | VXLAN migration completed and verified |

### Configuration Comparison

| Parameter | Before (IPIP) | After (VXLAN) |
|-----------|--------------|--------------|
| Encapsulation | IPIPAlways | VXLANCrossSubnet |
| MTU | 1440 | 1450 |
| Kernel Module | ipip.ko | vxlan.ko |
| Tunnel Interface | tunl0 | vxlan.calico |
| Overhead per packet | 20 bytes | 50 bytes |

The CNI network plugin is the backbone of pod communication in Kubernetes. This incident demonstrated that even a "routine" control plane upgrade can silently break the networking layer if the CNI plugin's dependencies are not validated. The fix — switching from IPIP to VXLAN — not only resolved the immediate outage but made the cluster more portable and resilient against kernel configuration differences in the future.

---

*This article is part of the Kubernetes Troubleshooting series. For more real-world incident analysis, see the Troubleshooting archive.*
