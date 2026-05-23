---
title: Baseline B - 3-Node k3s HA Installation with MetalLB VIP
project: K8s Thesis Prototype
date: 2026-05-22
type: installation-note
status: app-ready-with-metallb-vip
tags:
  - k3s
  - kubernetes
  - thesis
  - baseline-b
  - high-availability
  - alpine-linux
  - metallb
  - vip
---

# Baseline B - 3-Node k3s HA Installation with MetalLB VIP

This note documents the creation of **Baseline B**, a 3-node k3s high-availability-oriented prototype based on three Alpine Linux virtual machines.

The purpose of Baseline B is to move beyond the previous 2-node setup and create a more meaningful experimental architecture for node-failure, control-plane availability, and external-entrypoint availability experiments.

The latest version of Baseline B also introduces **MetalLB** to provide a single stable Virtual IP (VIP) for browser access to the prototype application.

---

## 1. Goal of Baseline B

Baseline B was created to improve the previous 2-node k3s prototype and provide a better environment for high-availability experiments.

Main goals:

```text
- remove the single control-plane dependency observed in Baseline A
- use three k3s server nodes with embedded etcd
- distribute the prototype workload across multiple nodes
- prepare the cluster for node-failure experiments
- introduce a stable external endpoint through MetalLB
```

Baseline B should be interpreted as an **HA-oriented experimental baseline**, not as a complete production-grade HA platform.

---

## 2. Architecture Summary

The cluster is composed of three Alpine Linux VMs. Each node acts as a k3s server node and participates in the control plane and embedded etcd datastore.

| Node | IP address | OS | Role |
|---|---:|---|---|
| `b-k3s-1` | `192.168.100.41` | Alpine Linux v3.23 | `control-plane,etcd` |
| `b-k3s-2` | `192.168.100.42` | Alpine Linux v3.23 | `control-plane,etcd` |
| `b-k3s-3` | `192.168.100.43` | Alpine Linux v3.23 | `control-plane,etcd` |

Additional endpoints:

| Element | Address | Purpose |
|---|---:|---|
| MetalLB VIP | `192.168.100.50` | Main browser endpoint for the prototype application |
| Rancher VM | `192.168.100.20` | External Rancher management server, accessed as `rancher.lab.local` |

Important distinction:

```text
The three Alpine VMs form the k3s cluster.
The Rancher server VM is external and is not part of the cluster.
Rancher agents still run inside the cluster in namespaces such as cattle-system and cattle-fleet-system.
MetalLB provides the stable application entrypoint after the cluster is running.
```

---

## 3. Preparation Before k3s Installation

The three nodes were created by cloning the original Alpine Linux VM. Before installing the new cluster, each clone was adjusted so that it had its own identity.

Preparation performed:

```text
- unique hostname for each VM
- unique static IP address
- consistent /etc/hosts mapping across the nodes
- removal of old k3s state inherited from the cloned VM
```

Reason:

```text
A cloned VM can contain old node identity, certificates, datastore files, runtime metadata, and network configuration.
Keeping that state would make the new nodes unreliable and could cause them to behave like copies of the old k3s node.
```

For this reason, Baseline B was created from clean k3s state rather than by reusing the previous cluster state from the cloned VM.

---

## 4. k3s HA Cluster Installation

The first node, `b-k3s-1`, was used to initialize a new k3s cluster with embedded etcd. The second and third nodes, `b-k3s-2` and `b-k3s-3`, were then joined as additional server nodes.

Final cluster result:

```text
b-k3s-1   Ready   control-plane,etcd   192.168.100.41
b-k3s-2   Ready   control-plane,etcd   192.168.100.42
b-k3s-3   Ready   control-plane,etcd   192.168.100.43
```

This confirmed that the cluster was running as a three-server k3s setup with embedded etcd.

A Hyper-V checkpoint was created at this stage to preserve the clean HA cluster before deploying the prototype application.

Checkpoint:

```text
baseline-b-clean-k3s-ha
```

---

## 5. Prototype Deployment

After the cluster was ready, the prototype images were imported on the nodes and the Kubernetes manifests were applied.

Application components:

```text
web-ui
api-backend
logic-simulator
data-generator
```

Initial result:

```text
api-backend       Running
data-generator    Running
logic-simulator   Running
web-ui            CrashLoopBackOff / Error
```

The cluster was healthy, but the Web UI was not starting correctly.

---

## 6. Web UI Service Issue and Fix

The Web UI failed because its Nginx configuration expected to reach `api-backend` through Kubernetes DNS.

Root cause:

```text
Kubernetes DNS resolves Service names, not Deployment names.
The api-backend Deployment existed, but there was no Service named api-backend.
```

Missing Services:

```text
api-backend
web-ui
```

Fix applied:

```text
- create Service for api-backend
- create Service for web-ui
- restart the web-ui deployment
- save both Service definitions as YAML manifests for reproducibility
```

After this correction, all application components reached a healthy state.

---

## 7. Application State Before MetalLB

Final application state before MetalLB:

```text
api-backend       3/3 Running
web-ui            3/3 Running
logic-simulator   1/1 Running
data-generator    1/1 Running
```

The application was reachable through the three node IPs:

```text
http://192.168.100.41
http://192.168.100.42
http://192.168.100.43
```

This confirmed that Traefik and the application ingress were working. However, this access model still forced the user to choose a specific node IP. For HA experiments, this was a limitation because the selected node could fail even while the rest of the cluster remained available.

This limitation motivated the introduction of MetalLB and the stable VIP:

```text
http://192.168.100.50
```

---

## 8. MetalLB VIP and Stable External Entrypoint

MetalLB was introduced after the application was already working in order to solve the external-entrypoint problem.

The objective was to avoid using a specific node IP as the browser entrypoint and to expose the prototype through one stable IP address:

```text
http://192.168.100.50
```

### 8.1 Why MetalLB was added

In the initial Baseline B configuration, Traefik was reachable through the node IPs:

```text
192.168.100.41
192.168.100.42
192.168.100.43
```

This was useful for proving that the application worked, but it was not a fully transparent access model for the user. The browser still had to target one node IP or manually choose another node IP.

MetalLB provides a Kubernetes-compatible way to assign a stable external IP address to a `LoadBalancer` Service in a bare-metal or on-premise-style environment. In this prototype, MetalLB assigns the external IP to the Traefik LoadBalancer Service.

Conceptually, the final request path becomes:

```text
User / browser on Windows
    -> MetalLB VIP 192.168.100.50
    -> Traefik Ingress / LoadBalancer Service
    -> web-ui Service
    -> web-ui Pods
    -> api-backend
    -> logic-simulator
    -> data-generator
```

Important distinction:

```text
Hyper-V / AlpineNAT provides the virtual LAN: 192.168.100.0/24
MetalLB announces the VIP inside that LAN: 192.168.100.50
Traefik receives and routes the HTTP request inside Kubernetes
```

Therefore, MetalLB does not replace Hyper-V networking and does not replace Traefik. It only provides a stable external IP for the Traefik `LoadBalancer` Service.

### 8.2 MetalLB location in the architecture

MetalLB is represented in the architecture in two ways:

```text
1. As a logical Kubernetes namespace:
   metallb-system
   - MetalLB controller
   - MetalLB speaker

2. As an external virtual IP used by the browser:
   MetalLB VIP: 192.168.100.50
```

The VIP should not be drawn as a pod inside the `metallb-system` namespace. The namespace contains the MetalLB components that manage and announce the VIP, while `192.168.100.50` is an external network address on the Hyper-V/AlpineNAT LAN.

In the diagram, the correct conceptual flow is:

```text
User / Browser
    -> MetalLB VIP 192.168.100.50
    -> Traefik Ingress / LoadBalancer Service
    -> Web UI
```

The MetalLB controller and speaker should be connected to the VIP with a separate management/announcement arrow, not with the main HTTP request flow.

### 8.3 Traefik after MetalLB

After MetalLB, Traefik should be interpreted as the component that receives browser traffic through the VIP.

Expected conceptual Traefik exposure:

```text
Traefik Service type: LoadBalancer
External IP: 192.168.100.50
```

This means that the browser no longer needs to access:

```text
http://192.168.100.41
http://192.168.100.42
http://192.168.100.43
```

The preferred access point becomes:

```text
http://192.168.100.50
```

### 8.4 Verification commands

Useful commands to verify MetalLB and Traefik exposure:

```sh
kubectl get ns
kubectl get pods -n metallb-system -o wide
kubectl get svc -n kube-system traefik -o wide
kubectl get ingress -n prototype -o wide
kubectl get ipaddresspool -n metallb-system
```

From Windows PowerShell:

```powershell
curl.exe -I http://192.168.100.50
```

Expected result:

```text
HTTP/1.1 200 OK
```

Browser test:

```text
http://192.168.100.50
```

### 8.5 What MetalLB solves and what it does not solve

MetalLB solves the stable-entrypoint problem:

```text
The user can access one stable IP address instead of choosing a node IP.
```

MetalLB does not automatically solve every availability problem:

```text
It does not make stateful databases highly available.
It does not replace application replication.
It does not remove the need for Kubernetes readiness/liveness checks.
It does not replace Traefik routing.
It does not make the Hyper-V host itself highly available.
```

For the thesis, the important interpretation is:

```text
- Baseline B improves the cluster/control-plane architecture through three embedded-etcd server nodes.
- MetalLB improves the external access model by introducing a stable VIP for Traefik.
- Together, they create a better baseline for node-failure experiments than Baseline A.
```

### 8.6 Current external access after MetalLB

Preferred browser endpoint:

```text
http://192.168.100.50
```

Legacy/direct node-IP access may still be useful for troubleshooting, but the MetalLB VIP should be used as the main endpoint for future experiments.

---

## 9. Hyper-V Checkpoint Status

The documented checkpoint that is known from this report is:

```text
baseline-b-app-ready
```

This checkpoint represents the application-ready state of Baseline B: the three-node k3s HA cluster was running, the prototype was deployed, the missing Service manifests had been added, and browser access was working.

After MetalLB was introduced, a more precise checkpoint name would be:

```text
baseline-b-app-ready-metallb-vip
```

However, the existence of this MetalLB-specific checkpoint has not been verified from inside the cluster VM. Therefore, it should be treated as a recommended checkpoint name unless it is confirmed from the Hyper-V host.

Recommended PowerShell command if the MetalLB-ready checkpoint still needs to be created:

```powershell
Checkpoint-VM -Name "b-k3s-1" -SnapshotName "baseline-b-app-ready-metallb-vip"
Checkpoint-VM -Name "b-k3s-2" -SnapshotName "baseline-b-app-ready-metallb-vip"
Checkpoint-VM -Name "b-k3s-3" -SnapshotName "baseline-b-app-ready-metallb-vip"
```

---

## 10. Current Verified State

The current checked state of Baseline B matches the intended architecture.

Cluster nodes:

```text
NAME      STATUS   ROLES                INTERNAL-IP
b-k3s-1   Ready    control-plane,etcd   192.168.100.41
b-k3s-2   Ready    control-plane,etcd   192.168.100.42
b-k3s-3   Ready    control-plane,etcd   192.168.100.43
```

Traefik exposure:

```text
NAMESPACE     NAME      TYPE           EXTERNAL-IP      PORTS
kube-system   traefik   LoadBalancer   192.168.100.50   80/TCP,443/TCP
```

Prototype ingress:

```text
NAMESPACE   NAME                CLASS     HOSTS   ADDRESS          PORTS
prototype   prototype-ingress   traefik   *       192.168.100.50   80
```

MetalLB address pool:

```text
NAMESPACE        NAME                 ADDRESSES
metallb-system   baseline-b-vip-pool  192.168.100.50/32
```

Application workload state:

```text
api-backend       3/3 Running   one pod per node
web-ui            3/3 Running   one pod per node
logic-simulator   1/1 Running
data-generator    1/1 Running
```

Summary:

| Area | Status |
|---|---|
| 3-node k3s cluster | OK |
| All nodes server/control-plane/etcd | OK |
| Embedded etcd HA baseline | OK |
| Traefik LoadBalancer | OK, `192.168.100.50` |
| MetalLB IPAddressPool | OK, `baseline-b-vip-pool` with `192.168.100.50/32` |
| Prototype ingress | OK, points to `192.168.100.50` |
| `api-backend` | OK, 3/3 |
| `web-ui` | OK, 3/3 |
| `logic-simulator` | OK, 1/1 |
| `data-generator` | OK, 1/1 |
| Rancher server | External to the cluster |
| Rancher agents | Present inside the cluster through Rancher namespaces |
| Browser access through MetalLB VIP | OK |
| Hyper-V checkpoint | `baseline-b-app-ready` documented; MetalLB-specific checkpoint recommended/needs host confirmation |

---

## 11. Important Notes

### 11.1 The Service YAML files were created only on `b-k3s-1`

The files:

```text
api-backend-service.yaml
web-ui-service.yaml
```

were created only on the filesystem of `b-k3s-1`.

This is fine for cluster functionality because the resources were applied to the Kubernetes cluster, which is shared by all nodes.

However, for reproducibility, the files should be saved in the project repository or copied to the other nodes.

Optional copy commands:

```sh
scp /root/k3s-prototype/k8s/api-backend-service.yaml root@192.168.100.42:/root/k3s-prototype/k8s/
scp /root/k3s-prototype/k8s/web-ui-service.yaml root@192.168.100.42:/root/k3s-prototype/k8s/

scp /root/k3s-prototype/k8s/api-backend-service.yaml root@192.168.100.43:/root/k3s-prototype/k8s/
scp /root/k3s-prototype/k8s/web-ui-service.yaml root@192.168.100.43:/root/k3s-prototype/k8s/
```

Better long-term solution:

```text
Commit the updated manifest files to the Git repository.
```

---

### 11.2 Control-plane HA is not the same as complete application HA

Baseline B improves the cluster/control-plane architecture because all three nodes are server/control-plane/etcd nodes.

MetalLB improves the external-entrypoint architecture because the browser can use one stable VIP:

```text
http://192.168.100.50
```

However, this still does not automatically mean that the complete application is production-grade highly available.

The following distinction should be kept clear in the thesis:

```text
3-node embedded etcd:
- improves Kubernetes control-plane and cluster-state availability

MetalLB VIP:
- improves external access continuity by avoiding a single node IP as the browser endpoint

Application replication:
- improves workload availability for stateless components

Database replication/failover:
- still required for stateful production workloads, but currently out of scope
```

Therefore, MetalLB closes one important gap in Baseline B, but it does not solve all availability concerns by itself.

---

## 12. Next Recommended Steps

The next experiments should start from this checkpoint:

```text
baseline-b-app-ready
```

Recommended experimental sequence:

```text
1. Replica distribution analysis
2. Verify and document MetalLB VIP access through 192.168.100.50
3. Web UI pod deletion on Baseline B
4. Node failure while using the MetalLB VIP as browser endpoint
5. Server/control-plane node failure on Baseline B
6. Comparison with Baseline A 2-node results
```

The most important comparison is:

```text
Baseline A: main node failure caused endpoint timeout and exposed a critical point.
Baseline B: test whether 3 server nodes reduce or eliminate that critical point.
```

---

## 13. Useful Commands

### Cluster state

```sh
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

### Prototype state

```sh
kubectl get all -n prototype
kubectl get pods -n prototype -o wide
kubectl get svc -n prototype
kubectl get ingress -n prototype -o wide
```

### Images

```sh
k3s ctr images ls | grep -E "web-ui|api-backend|logic-simulator|data-generator"
```

### Restart Web UI

```sh
kubectl rollout restart deployment/web-ui -n prototype
```

### Check Web UI logs

```sh
kubectl logs -n prototype deployment/web-ui --previous
```

### MetalLB

```sh
kubectl get pods -n metallb-system -o wide
kubectl get svc -n kube-system traefik -o wide
kubectl get ingress -n prototype -o wide
kubectl get ipaddresspool -n metallb-system
```

### Browser test

Preferred endpoint after MetalLB:

```text
http://192.168.100.50
```

Legacy node-IP troubleshooting endpoints:

```text
http://192.168.100.41
http://192.168.100.42
http://192.168.100.43
```

---

# Final Conclusion

Baseline B has been successfully created.

It consists of a three-node k3s HA-oriented cluster where every node is a server/control-plane/etcd node. The prototype application is deployed and reachable from the browser. Initially, the application was reachable through the node IPs. The updated Baseline B architecture also introduces MetalLB, which exposes Traefik through the stable VIP `192.168.100.50`.

The missing Service definitions for `api-backend` and `web-ui` were identified, fixed, and saved as YAML manifests. MetalLB should now be treated as the preferred external-entrypoint mechanism for future experiments.

This baseline is ready for the next experimental phase focused on high availability, node failure, stable external access through the MetalLB VIP, and comparison against the original 2-node baseline.
