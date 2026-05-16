> ⚠️ **Mirror.** Primary repository: [git.digtvbg.com](https://git.digtvbg.com/ivangotoy/homelab-k8s)
> Development, issues, and PRs happen there. The GitHub repo is read-only.

# homelab-k8s

4-node bare-metal Kubernetes cluster running on Intel N100 mini-PCs with CachyOS.
Production-grade tooling: Calico CNI, Linkerd service mesh, cert-manager, MetalLB,
NFS storage, Trivy security scanning, and automated node reboots via kured.

---

## Cluster

| Node | Role | IP | CPU | RAM | Storage |
|---|---|---|---|---|---|
| kube-controller | Control plane | 192.168.1.10 | Intel N100 | 16GB | 236GB NVMe (btrfs) |
| node-01 | Worker | 192.168.1.9 | Intel N100 | 16GB | 236GB NVMe (btrfs) |
| node-02 | Worker | 192.168.1.11 | Intel N100 | 16GB | 236GB NVMe (btrfs) |
| node-03 | Worker | 192.168.1.12 | Intel N100 | 16GB | 236GB NVMe (btrfs) |

- **OS:** CachyOS (Arch-based)
- **Kernel:** 6.19.9-1-cachyos-server-lto
- **Kubernetes:** v1.35.2
- **Container runtime:** containerd 2.2.2

---

## Storage

NFS server hosted on OpenWrt router with a 10TB USB HDD.
Two storage provisioners running in parallel:

- `csi-driver-nfs` — CSI-compliant NFS driver for dynamic PVC provisioning
- `nfs-subdir-external-provisioner` — legacy subdir provisioner

---

## Helm Releases

| Release | Namespace | Chart | Version |
|---|---|---|---|
| calico | tigera-operator | tigera-operator | v3.31.4 |
| cert-manager | cert-manager | cert-manager | v1.20.0 |
| docker-registry | default / docker-registry | docker-registry | 3.0.0 |
| envoy-gateway | envoy-gateway | envoy-gateway | 1.8.0 |
| kubewall | kubewall-system | kubewall | 0.0.17 |
| kured | kured | kured | 5.11.0 |
| linkerd-control-plane | linkerd | linkerd-control-plane | edge-26.3.3 |
| linkerd-viz | linkerd-viz | linkerd-viz | edge-26.3.3 |
| metallb | metallb-system | metallb | 0.15.3 |
| metrics-server | kube-system | metrics-server | 3.13.0 |
| csi-driver-nfs | kube-system | csi-driver-nfs | 4.13.1 |
| nfs-subdir | nfs-provisioner | nfs-subdir-external-provisioner | 4.0.18 |
| trivy-operator | trivy-system | trivy-operator | 0.32.1 |

---

## Key Components

### Networking
- **CNI:** Calico v3.31.4 (via Tigera operator) — BGP-capable, NetworkPolicy enforcement
- **Service mesh:** Linkerd edge-26.3.3 — mTLS, traffic policy, observability
- **Load balancer:** MetalLB v0.15.3 — bare-metal LB via L2/BGP
- **Ingress:** ingress-nginx 1.15.1

### Storage
- **CSI NFS driver** — dynamic PVC provisioning from OpenWrt/NFS
- **NFS subdir provisioner** — legacy workloads

### Security
- **cert-manager** v1.20.0 — automated TLS certificate lifecycle
- **Trivy operator** — continuous vulnerability scanning of workloads

### Operations
- **kured** — watches `/var/run/reboot-required` sentinel, drains and reboots nodes
  one at a time automatically after kernel/package updates
- **metrics-server** — resource metrics for HPA and `kubectl top`
- **kubewall** — lightweight Kubernetes dashboard

### Registry
- Private Docker registry running inside the cluster for local image hosting

---

## Automated Node Reboots

kured runs as a DaemonSet on all nodes. When a node requires a reboot
(kernel update, package update), a sentinel file is created at
`/var/run/reboot-required`. kured detects this, cordons and drains the node,
reboots it, and waits for it to rejoin before moving to the next node.

Sentinel creation is handled by custom pacman hooks + needrestart integration
on CachyOS nodes.

---

## Node Management

Nodes are managed via Ansible — see
[ansible-playbooks](https://github.com/ivangotoy/ansible-playbooks):

- Rolling package upgrades via `paru` (serial, one node at a time)
- Performance tuning: CPU governor, THP, network buffers, containerd config
- Mirror refresh via `cachyos-rate-mirrors`
