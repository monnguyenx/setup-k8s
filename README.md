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

### 2.2. Install Containerd
```
sudo apt update
sudo apt install -y containerd
```

1. Create standard configuration for K8s
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit line:
```
SystemdCgroup = true
```
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart:
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 2.3. Install Kubeadm/Kubelet/Kubectl
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
```

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2.4. Initialize cluster
```
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16
```

Wait 1-2 minutes, and eventually will see the join token message → 
    If you want another node to join, add this token to that node.

### 2.5. Setup Kubect for User
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check:
```
kubectl get nodes
```
➡️ Node is NotReady: True (no CNI yet)

### 2.6. Install Flannel (CNI)
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Wait:
```
kubectl get pods -n kube-system -w
```

When:
- flannel → Running
- coredns → Running

➡️ DONE

### 2.7. Allow Pod running in Control-plane
If have 1-node K8s (Control-plane + worker), you must allow pod running in Control-plane

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 2.8. Final Check
```
kubectl get nodes
kubectl get pods -A
```

### 2.9. Desired results
```
STATUS: Ready
```

## 3. Install Argo CD
Steps:
### 3.1. Create Namespace ArgoCD
```
kubectl create namespace argocd
```

### 3.2. Install ArgoCD Official Manifest
👉 Use the stable version (The standard version used by companies)

```
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

⏳ Wait 1-2 minutes

### 3.3. Check Pod
```
kubectl get pods -n argocd
```

✅ If OK:
```
argocd-server-xxxx        1/1 Running
argocd-repo-server-xxxx   1/1 Running
argocd-application-controller-0 1/1 Running
argocd-redis-xxxx         1/1 Running
```

### 3.4. Expose ArgoCD UI
Edit service:
```
kubectl edit svc argocd-server -n argocd
```

Change:
```
type: ClusterIP
```

➡️ Result:
```
type: NodePort
```

Save.

Check Port:
```
kubectl get svc -n argocd argocd-server
```

Example:
```
argocd-server   NodePort   10.104.207.114   <none>        80:30224/TCP,443:32709/TCP   50m
```

👉 UI:
```
https://<IP-SERVER>:32709
```

### 3.5. Get Login Password
Get password:
```
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Default User:
```
admin
```

📌 Login success → Change password

### 3.6. Install ArgoCD CLI
```
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

Login:
```
argocd login <IP-SERVER>:<PORT> --username admin --password <PASSWORD> --insecure
```

### 3.7. Change password
```
argocd account update-password --account admin --current-password <PASSWORD> --new-password <NEW_PASSWORD>
```
