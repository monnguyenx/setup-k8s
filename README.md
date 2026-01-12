# Kubernetes + Argo CD (GitOps from Scratch)

This repository documents step-by-step how to:
- Install Kubernetes using kubeadm
- Set up cluster networking
- Install and configure Argo CD
- Deploy applications using GitOps

---

## 1. Prerequisites
- Ubuntu 20.04+
- Docker / Containerd
- kubeadm, kubelet, kubectl
- At least 2GB RAM

## 2. Install Kubernetes
Steps:
### 2.1. Prepare the system
1. Disables swap ()
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

2. Load kernel module & sysctl
```
sudo modprobe br_netfilter
sudo modprobe overlay
```

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```