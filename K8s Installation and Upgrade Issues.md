# Kubernetes Issues and Upgrade Notes

## Cluster Information

### Current Cluster Version

```bash
oc get nodes -o wide
```

### Output

```text
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master01   Ready    control-plane   93d   v1.30.5   192.168.1.51   Ubuntu 24.04.2 LTS   6.11.0-21-generic   containerd://1.7.24
worker01   Ready    <none>          93d   v1.30.5   192.168.1.50   Ubuntu 24.04.1 LTS   6.11.0-21-generic   containerd://1.7.24
worker02   Ready    <none>          92d   v1.30.5   192.168.1.52   Ubuntu 24.04.2 LTS   6.11.0-26-generic   containerd://1.7.27
```

---

# APT Repository GPG Key Issue

## Problem

Running:

```bash
apt-get update
```

returns:

```text
Err:5 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb InRelease
  The following signatures were invalid:

  EXPKEYSIG 234654DA9A296436
  isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
```

---

## Root Cause

The Kubernetes package repository signing key has expired or is no longer trusted.

APT cannot validate package authenticity and therefore refuses to update repository metadata.

---

# Fix Kubernetes Repository Key

## Create Keyring Directory

```bash
sudo mkdir -p /etc/apt/keyrings
```

## Download Current Kubernetes Repository Key

```bash
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
| sudo gpg --dearmor \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

## Configure Repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## Refresh Package Metadata

```bash
sudo apt-get update
```

---

# Kubernetes Upgrade Path

## Important Upgrade Rule

Kubernetes does **not** support skipping minor versions.

### Supported Upgrade Sequence

```text
v1.30.x
   ↓
v1.31.x
   ↓
v1.32.x
   ↓
v1.33.x
```

For a cluster currently running v1.30.5:

```text
1.30.5 → 1.31.x
1.31.x → 1.32.x
1.32.x → 1.33.x
```

Each step must be completed successfully before proceeding to the next version.

---

# Upgrade Workflow Overview

For each minor version upgrade:

1. Upgrade `kubeadm`
2. Upgrade control plane
3. Upgrade `kubelet` and `kubectl`
4. Restart kubelet
5. Upgrade worker nodes
6. Validate cluster health

---

# Configure Repository for Target Version

Example for Kubernetes v1.33:

```bash
sudo rm -f /etc/apt/sources.list.d/kubernetes.list

sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
EOF

sudo apt-get update
```

---

# Upgrade Control Plane

## Upgrade kubeadm

```bash
sudo apt-get install -y kubeadm=1.31.0-1.1
```

## Verify Upgrade Plan

```bash
sudo kubeadm upgrade plan
```

## Apply Upgrade

```bash
sudo kubeadm upgrade apply v1.31.0
```

---

# Upgrade kubelet and kubectl

```bash
sudo apt-get install -y \
kubelet=1.31.0-1.1 \
kubectl=1.31.0-1.1
```

Restart kubelet:

```bash
sudo systemctl restart kubelet
```

---

# Upgrade Worker Nodes

## Drain Worker

```bash
kubectl drain <node-name> --ignore-daemonsets
```

## Upgrade kubeadm

```bash
sudo apt-get install -y kubeadm=1.31.0-1.1
```

## Upgrade Node Configuration

```bash
sudo kubeadm upgrade node
```

## Upgrade kubelet and kubectl

```bash
sudo apt-get install -y \
kubelet=1.31.0-1.1 \
kubectl=1.31.0-1.1
```

## Restart kubelet

```bash
sudo systemctl restart kubelet
```

## Uncordon Worker

```bash
kubectl uncordon <node-name>
```

---

# Repeat for Remaining Versions

After completing v1.31.x:

```text
v1.31.x → v1.32.x
```

Repeat the same procedure.

Then:

```text
v1.32.x → v1.33.x
```

Repeat again.

---

# Verify Available Package Versions

```bash
apt-cache madison kubeadm
```

Example:

```bash
apt-cache madison kubeadm | grep 1.33
```

---

# Offline Upgrade Preparation

If worker nodes do not have internet access, download required packages and images from the control plane.

---

## Configure Repository for v1.31

```bash
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
| sudo gpg --dearmor \
-o /usr/share/keyrings/kubernetes-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt-get update
```

---

## Download Kubernetes Packages

```bash
apt-get download \
kubeadm=1.31.10-00 \
kubelet=1.31.10-00 \
kubectl=1.31.10-00
```

---

# Download Kubernetes Images

## Pull Images

```bash
kubeadm config images pull \
--kubernetes-version v1.31.10
```

Expected images:

```text
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
coredns
pause
etcd
```

---

## Export Images

```bash
images=$(kubeadm config images list \
--kubernetes-version v1.31.10)

for img in $images; do
    sanitized=$(echo $img | sed 's|/|_|g' | sed 's|:|_|g')
    sudo ctr -n k8s.io images export \
    ${sanitized}.tar $img
done
```

---

# Transfer Files to Worker Nodes

## Copy Packages

```bash
scp *.deb worker-node:/tmp/k8s/
```

## Copy Images

```bash
scp *.tar worker-node:/tmp/k8s/
```

---

# Install Packages on Worker Nodes

```bash
sudo dpkg -i /tmp/k8s/*.deb
```

---

# Import Images

```bash
for tarball in /tmp/k8s/*.tar; do
    sudo ctr -n k8s.io images import $tarball
done
```

---

# Upgrade Worker Nodes

```bash
sudo kubeadm upgrade node
```

```bash
sudo systemctl daemon-reexec
sudo systemctl restart kubelet
```

---

# Post-Upgrade Validation

## Verify Nodes

```bash
kubectl get nodes
```

## Verify Kubernetes Version

```bash
kubectl version
```

## Verify kubeadm Version

```bash
kubeadm version
```

## Verify kubelet Version

```bash
kubelet --version
```

---

# Compatibility Checks

Before upgrading:

- Verify the CNI plugin supports the target Kubernetes version.
- Review Kubernetes release notes.
- Validate CSI drivers and storage classes.
- Check ingress controller compatibility.
- Verify monitoring and logging components.

---

# Backup Recommendations

Before any control-plane upgrade:

## Backup etcd

```bash
ETCDCTL_API=3 etcdctl snapshot save etcd-backup.db
```

## Verify Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot status etcd-backup.db
```

---

# Quick Upgrade Checklist

| Task | Status |
|--------|--------|
| Backup etcd | ☐ |
| Fix Kubernetes repository key | ☐ |
| Validate CNI compatibility | ☐ |
| Upgrade kubeadm | ☐ |
| Upgrade control plane | ☐ |
| Upgrade kubelet and kubectl | ☐ |
| Upgrade worker nodes | ☐ |
| Verify node status | ☐ |
| Verify workloads | ☐ |
| Repeat for next minor version | ☐ |
| Reach target version (v1.33.x) | ☐ |

---
