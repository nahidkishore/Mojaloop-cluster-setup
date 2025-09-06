

# 🚀 Mojaloop Deployment on K3s (v1.29 HA Cluster)

This guide will walk you through deploying a **Highly Available (HA) K3s Kubernetes Cluster** with **3 master nodes + 3 worker nodes**, and then installing **Mojaloop** with **Helm**.

---

## 📌 1. Prerequisites

* **6 Ubuntu 24.04 EC2 instances**

  * 3 Masters (control-plane)
  * 3 Workers (compute)
  * Each: minimum 2 vCPU, 4GB RAM, 20GB storage
* **Security groups** open for:

  * `6443` (Kubernetes API)
  * `2379–2380` (etcd)
  * `10250–10252` (kubelet, scheduler, controller-manager)
  * `8472/UDP` (Flannel VXLAN)
  * `30000–32767` (NodePort services)
* **Root / sudo access** to all nodes
* Tools installed: `curl`, `wget`, `jq`, `helm`

---

## 🟢 2. Prepare All Nodes

Run the following **on all 6 nodes**:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget apt-transport-https gnupg2 software-properties-common lsb-release -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

---

## 🟢 3. Install K3s on Master-1 (Cluster Init)

On **Master-1**:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" sh -s - server \
  --cluster-init \
  --tls-san <LOAD_BALANCER_IP or MASTER-1_PUBLIC_IP> \
  --write-kubeconfig-mode 644
```

Check:

```bash
sudo systemctl status k3s
kubectl get nodes
```

---

## 🟢 4. Add Master-2 & Master-3

First, get the join token from Master-1:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Then on **Master-2 & Master-3**:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" \
  K3S_URL="https://<MASTER-1_PRIVATE_IP>:6443" \
  K3S_TOKEN="<PASTE_TOKEN>" sh -s - server \
  --tls-san <LOAD_BALANCER_IP or MASTER-1_PUBLIC_IP> \
  --write-kubeconfig-mode 644
```

---

## 🟢 5. Configure `kubectl`

On Master-1:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Verify:

```bash
kubectl get nodes
kubectl cluster-info
```

---

## 🟢 6. Add Worker Nodes

Get token from Master-1 again:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On **Worker-1, Worker-2, Worker-3**:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" \
  K3S_URL="https://<MASTER-1_PRIVATE_IP>:6443" \
  K3S_TOKEN="<PASTE_TOKEN>" sh -s -
```

Verify:

```bash
kubectl get nodes -o wide
```

✅ Now you should see **3 masters + 3 workers** in the cluster.

---

## 🟢 7. Install Helm

On Master-1:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

---

## 🟢 8. Install Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
kubectl get pods -n ingress-nginx
```

---

## 🟢 9. Add Mojaloop Helm Repo

```bash
helm repo add mojaloop https://mojaloop.io/helm/repo/
helm repo update
```

---

## 🟢 10. Deploy Backend Dependencies

```bash
helm install backend mojaloop/example-mojaloop-backend --namespace demo --create-namespace
kubectl get pods -n demo
```

---

## 🟢 11. Deploy Mojaloop Core

```bash
helm install moja mojaloop/mojaloop --namespace demo --create-namespace
```

Check status:

```bash
helm -n demo list
kubectl get pods -n demo
```

You should see multiple Mojaloop components running:

* **Account Lookup Service (ALS)**
* **Central Ledger**
* **Quoting Service**
* **Transaction Request Service**
* **Testing Toolkit**
* **Simulators**

---



# command line History


---

# 📌 Mojaloop Deployment with K3s (Command List)

```bash
# 🔹 Step 1: System Preparation (all nodes: 3 masters + 3 workers)
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget apt-transport-https gnupg2 software-properties-common lsb-release -y

# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

---

```bash
# 🔹 Step 2: Install K3s on Master-1
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" sh -s - server \
  --cluster-init \
  --tls-san <LOAD_BALANCER_IP_or_MASTER-1_PUBLIC_IP> \
  --write-kubeconfig-mode 644

# Verify
sudo systemctl status k3s
kubectl get nodes
```

---

```bash
# 🔹 Step 3: Join Master-2 & Master-3
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" \
  K3S_URL="https://<MASTER-1_PRIVATE_IP>:6443" \
  K3S_TOKEN="$(sudo cat /var/lib/rancher/k3s/server/node-token)" \
  sh -s - server \
  --tls-san <LOAD_BALANCER_IP_or_MASTER-1_PUBLIC_IP> \
  --write-kubeconfig-mode 644
```

---

```bash
# 🔹 Step 4: Configure kubeconfig (on any master)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

kubectl get nodes
kubectl cluster-info
```

---

```bash
# 🔹 Step 5: Join Worker Nodes (worker-1, worker-2, worker-3)
# First, fetch token from master-1:
sudo cat /var/lib/rancher/k3s/server/node-token

# Then on each worker node:
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.29.5+k3s1" \
  K3S_URL="https://<MASTER-1_PRIVATE_IP>:6443" \
  K3S_TOKEN="<PASTE_TOKEN>" \
  sh -s -
```

---

```bash
# 🔹 Step 6: Verify Cluster
kubectl get nodes -o wide
kubectl get pods -A
```

---

```bash
# 🔹 Step 7: Install Helm (on master-1)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

---

```bash
# 🔹 Step 8: Install Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

kubectl get pods -n ingress-nginx
```

---

```bash
# 🔹 Step 9: Add Mojaloop Repo
helm repo add mojaloop https://mojaloop.io/helm/repo/
helm repo update
```

---

```bash
# 🔹 Step 10: Deploy Backend Dependencies
helm install backend mojaloop/example-mojaloop-backend --namespace demo --create-namespace
kubectl get pods -n demo
```

---

```bash
# 🔹 Step 11: Deploy Mojaloop Core
helm install moja mojaloop/mojaloop --namespace demo --create-namespace

helm -n demo list
kubectl get pods -n demo
```

---

