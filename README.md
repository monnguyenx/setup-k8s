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

## 4. Install Helm
- Helm is a package manager for Kubernetes that enables reusable
and configurable application deployments.

- Helm charts allow defining Kubernetes manifests once and reusing them
across multiple environments by providing different configuration values.

Install Helm:

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Check:
```
helm version
```

## 5. Install Cluster Infrastructure (Helm-based)

Install foundation components for the cluster before deploying the app

### 5.1. Infrastructure Components Overview

Before deploying applications with Argo CD, the Kubernetes cluster must have
basic infrastructure components installed.

These components provide storage, networking, monitoring, and management
capabilities that most production workloads depend on.

#### Metrics Server
- Collect CPU / Memory usage from kubelet
- Required for:
  - ``` kubectl top ```
  - Horizontal Pod Autoscaler (HPA)
- If not present → HPA does not function

#### Longhorn
- Distributed block storage cho Kubernetes
- Provide PersistentVolume (PVC)
- Required for:
  - Database
  - StatefulSet
- If not present → PVC stuck in pending

#### NGINX Ingress Controller
- HTTP/HTTPS ingress for cluster
- Route traffic from outside to the Service
- Required for:
  - Web UI
  - API Gateway
- If not present → Ingress does not work

#### Portainer (Optional)
- Web UI manages Kubernetes
- Suitable for:
  - Lab
  - Demo
  - Debug quickly

#### Rancher (Optional/ Advanced)
- Kubernetes management platform
- Management:
  - Users/RBAC
  - Multiple clusters
- Often used in enterprises

#### → The next step will be installation instructions
### 5.2. Install Metrics Server
#### Step 1. Pull repo metric-server in GitLab

#### Step 2. Run command:
```
helm install metrics-server ./helm/metrics-server \
  --namespace kube-system \
  --create-namespace
```

#### Note: Important settings in values.yaml
- apiService.create: true → Metrics are provided to the API
- rbac.create: true → Full Permissions
- serviceAccount.create: true → ServiceAccount is created
- replicas: 1 → Deploy 1 pod

### 5.2. Install Longhorn
#### Step 1. Pull repo longhorn in GitLab

#### Step 2. Run CCommand
```
helm install longhorn ./helm/longhorn \
  --namespace longhorn-system \
  --create-namespace
```

#### Step 3. Check Install
Check pod:
```
kubectl get pods -n longhorn-system
```

Check StorageClass:
```
kubectl get storageclass
```

Access Longhorn UI (dashboard):
```
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

#### Note: Important settings in values.yaml
- defaultSettings.defaultDataPath: ~ → Specify the data storage directory on node (default: /var/lib/longhorn/)
- persistence.defaultClassReplicaCount=2 → Number of Longhorn data volume copies on different nodes
- defaultSettings.storageMinimalAvailablePercentage=25 → Minimum free space threshold on each node storing Longhorn data
- persistence.defaultFsType=ext4 → Filesystem is used when Longhorn first formats a volume
- ingress → Create an Ingress object for NGINX controller to route traffic from host via path "/" to Longhorn UI pod, with basic authentication protection

### 5.3. NGINX Ingress Controller
#### Step 1. Pull repo nginx-ingress-controller in GitLab

#### Step 2. Run Command
```
helm install nginx-ingress-controller ./helm/nginx-ingress-controller \
  --namespace ingress-nginx \
  --create-namespace
```

#### Step 3. Check Install
Pod Running:
```
kubectl get pods -n ingress-nginx
```

IP Service:
```
kubectl get svc -n ingress-nginx
```

Ingress is working:
```
kubectl get ingress -n longhorn-system
```

#### Note: Important settings in values.yaml
On premise:
- kind: DaemonSet
- useHostPort: false 
- service.type: NodePort
- ingressClassResource.default: true
- replicaCount: 2

Cloud:
- kind: Deployment
- replicaCount=3
- service.type=LoadBalancer

### 5.4. Portainer
#### Step 1. Pull repo portainer in GitLab

#### Step 2. Run Command
```
helm install portainer ./helm/portainer \
  --namespace portainer \
  --create-namespace
```

#### Step 3. Check Install
Pod Running:
```
kubectl get pods -n portainer
```

IP Service:
```
kubectl get svc -n portainer
```

Ingress is working:
```
kubectl get ingress -n portainer
```

#### Note: Important settings in values.yaml
- type: NodePort
- ingress.enabled: true
- ingress.ingressClassName: nginx
- ingress.hosts[0].host: portainer.vti-solutions.vn
- persistence
- replicaCount
- resources

### 5.5. Rancher
#### Step 1. Pull repo rancher in GitLab

#### Step 2. Run Command
```
helm install rancher ./helm/rancher \
  --namespace cattle-system \
  --create-namespace
```

#### Step 3. Check Cert-manager
```
kubectl get ns | grep cert-manager
kubectl get crd | grep cert
```

If is not, install cert-manager:
```
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
```

#### Step 4. Check Install
Pod Running:
```
kubectl get pods -n cattle-system
```

IP Service:
```
kubectl get svc -n cattle-system rancher
```

Ingress is working:
```
kubectl get ingress -n cattle-system
```

Check certificate:
```
kubectl get cert -n cattle-system
```

#### Step 5. Rancher Web
Get Rancher URL:
```
kubectl get svc -n cattle-system
```

Get password:
```
kubectl get secret --namespace cattle-system bootstrap-secret -o jsonpath='{.data.bootstrapPassword}' | base64 -d
```

#### Note: Important settings in values.yaml
- hostname
- ingress
- ingress.tls.source
- ingress.tls.secretName
- service
- replicas
- priorityClassName

## 5. Config repository
### 5.1. Tạo repo CICD Configs
```text
.
├── argo-cd
│   ├── bootstrap.yaml
│   └── project.yaml
├── bootstrap
│   ├── Chart.yaml
│   ├── templates
│   │   ├── services
│   │   │   ├── attribute-service.yaml
│   │   │   ├── auth-service.yaml
│   │   │   ├── cost-center-service.yaml
│   │   │   ├── datasync-service.yaml
│   │   │   ├── item-service.yaml
│   │   │   ├── item-stock-service.yaml
│   │   │   ├── master-data-service.yaml
│   │   │   ├── mesx-grand.yaml
│   │   │   ├── request-service.yaml
│   │   │   ├── sale-service.yaml
│   │   │   ├── ticket-service.yaml
│   │   │   ├── user-service.yaml
│   │   │   ├── warehouse-layout-service.yaml
│   │   │   └── warehouse-service.yaml
│   │   └── tools
│   │       ├── kafka.yaml
│   │       ├── kong.yaml
│   │       ├── nats.yaml
│   │       └── redis.yaml
│   └── values.yaml
├── databases
├── manifests
│   ├── services
│   │   ├── attribute-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── auth-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── cost-center-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── datasync-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── item-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── item-stock-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── master-data-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── mesx-grand
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── service.yaml
│   │   ├── request-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── sale-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── ticket-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── user-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   ├── warehouse-layout-service
│   │   │   ├── config.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── secret.yaml
│   │   │   └── service.yaml
│   │   └── warehouse-service
│   │       ├── config.yaml
│   │       ├── deployment.yaml
│   │       ├── secret.yaml
│   │       └── service.yaml
│   └── tools
│       ├── kafka
│       │   ├── Chart.lock
│       │   ├── charts
│       │   ├── Chart.yaml
│       │   ├── README.md
│       │   ├── templates
│       │   └── values.yaml
│       ├── kong
│       │   ├── Chart.lock
│       │   ├── charts
│       │   ├── Chart.yaml
│       │   ├── crds
│       │   ├── README.md
│       │   ├── templates
│       │   └── values.yaml
│       ├── nats
│       │   ├── Chart.lock
│       │   ├── charts
│       │   ├── Chart.yaml
│       │   ├── README.md
│       │   ├── templates
│       │   └── values.yaml
│       └── redis
│           ├── Chart.lock
│           ├── charts
│           ├── Chart.yaml
│           ├── README.md
│           ├── templates
│           ├── values.schema.json
│           └── values.yaml


Đây là cấu trúc hình cây
## 6. Config terraform
## 7. Config Jenkins job
## 8. ArgoCD
## 9. Platform services
## 10. Application services
