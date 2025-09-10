

# HA Kubernetes Cluster Setup (Production-Grade)

This repository contains a complete step-by-step guide to deploy a **Highly Available Kubernetes Cluster** using `kubeadm`, `HAProxy` load balancer, and the `Calico` CNI plugin.

It is designed to replicate **real-world production environments**, following industry best practices for **resiliency, fault tolerance, and scalability**.

---

## 📖 About this Documentation

This documentation has been authored by **\[Your Name / Nahidul Islam]**, based on real-world, production-grade deployment experience as a **DevOps & Cloud Engineer**.
It provides both **conceptual clarity** and **hands-on practical steps** so that engineers can set up, validate, and operate a production-ready Kubernetes cluster.

---

## 🎯 Objectives

* Build an **HA Kubernetes cluster** with:

  * 3 × Control Plane Nodes
  * 3 × Worker Nodes
  * 1 × Load Balancer (HAProxy)
  * 3 × External etcd nodes(optional)
* Ensure **resilience**: cluster stays healthy even if one control-plane or etcd node fails.
* Provide a **reference architecture** for teams looking to deploy Kubernetes outside managed cloud environments.

---




# HA Kubernetes Cluster Setup | Highly Available K8 Cluster

* **Topology**: 1 × Load-Balancer (HAProxy) + 3 × Control-plane (masters) + 3 × Workers
* **Kubernetes**: **latest stable** (as of Sep 2025 → **v1.34**). ([Kubernetes][1])
* **CNI**: **Calico latest stable** (use v3.30.x manifests — example uses v3.30.3). ([docs.tigera.io][2], [GitHub][3])


---

## Quick checklist (before you begin)

* All nodes run Ubuntu (22.04/24.04 ok) with root/sudo access.
* Time in sync (install `ntp`/`chrony` if needed).
* Ports open in firewall / security groups between nodes (see earlier SG list — must allow `6443`, `2379-2380`, `10250`, `30000-32767`, Calico ports, SSH).
* LB VM is reachable by masters & workers on port **6443**.
* Use **3 masters** (etcd quorum safe). Good — we’re using 3 masters here.

# Install HAProxy on the single LB VM

### Install HAProxy:
```bash

sudo apt-get update
sudo apt-get install -y haproxy
```

## Configure HAProxy: Edit the HAProxy configuration file (/etc/haproxy/haproxy.cfg):
```bash
sudo vim /etc/haproxy/haproxy.cfg

```

## Add the following configuration:

```bash

frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <MASTER1_IP>:6443 check
    server master2 <MASTER2_IP>:6443 check


```


```bash

frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 54.85.195.111:6443 check
    server master2 44.222.254.44:6443 check
    server master3 100.26.35.205:6443 check

    
```
### Restart HAProxy:
```bash

sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

# 2) Prep all nodes (LB, masters, workers)

Run on **every node**:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release software-properties-common jq
# disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
# kernel modules & sysctl
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

```

# 3 Step 3: Install CRI-O (Container Runtime Interface) on All master & worker Nodes
Run these commands on both Master and Worker Nodes.

Add CRI-O Repository and Install CRI-O:

```bash
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o
 #Start and Enable CRI-O:

sudo systemctl daemon-reload
sudo systemctl enable crio --now

```

# Step 4: Install Kubernetes Components (kubeadm, kubelet, kubectl)

### Run these commands on both Master and Worker Nodes.

Add Kubernetes APT Repository:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
# Install kubeadm, kubelet, and kubectl:

sudo apt-get update -y
sudo apt-get install -y kubelet="1.34.0-*" kubectl="1.34.0-*" kubeadm="1.34.0-*"
sudo apt-mark hold kubelet kubeadm kubectl
# Enable and Start kubelet:

sudo systemctl enable --now kubelet

```

# Step 5: Initialize the First Master Node

### Run the following commands on the first Master Node.

#### Pull Required Images:
```bash
sudo kubeadm config images pull


```

### Initialize the first master node:
```bash

sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_IP:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
for example: 

```bash
sudo kubeadm init --control-plane-endpoint "54.167.6.148:6443" --upload-certs --pod-network-cidr=192.168.0.0/16

  ```

## you can see this type of output:

```bash
You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join 54.167.6.148:6443 --token jxi0rw.q7ljkiipq21zwqub \
	--discovery-token-ca-cert-hash sha256:03041be8bf92ef8748792c9f1d30230b176ab579457dc0bc443f3a5c79b5d690 \
	--control-plane --certificate-key 1043d06b2f5cf0a2f8b3cab4a8b2c6c593edacba176a2abffe34bdefd7c16d18

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 54.167.6.148:6443 --token jxi0rw.q7ljkiipq21zwqub \
	--discovery-token-ca-cert-hash sha256:03041be8bf92ef8748792c9f1d30230b176ab579457dc0bc443f3a5c79b5d690

```

### Set up kubeconfig for the first master node:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Install Calico network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.5/manifests/calico.yaml

```


# Step 6: Join the Second & third Master Node

Get the join command and certificate key from the first master node:
```bash

kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
```

## Run the join command on the second master node:
```bash

sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key>
```
## Set up kubeconfig for the second master node:
```bash

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# Step 7: Join the Worker Nodes
Get the join command from the first master node:
```bash

kubeadm token create --print-join-command
```
### Run the join command on each worker node:
```bash
sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
## Output

```bash


ubuntu@ip-172-31-22-30:~$ kubectl get nodes
NAME               STATUS   ROLES           AGE     VERSION
ip-172-31-17-28    Ready    control-plane   3m55s   v1.34.0
ip-172-31-17-42    Ready    <none>          2m56s   v1.34.0
ip-172-31-20-164   Ready    <none>          2m38s   v1.34.0
ip-172-31-21-187   Ready    <none>          3m14s   v1.34.0
ip-172-31-22-30    Ready    control-plane   16m     v1.34.0
ip-172-31-27-126   Ready    control-plane   4m53s   v1.34.0
ubuntu@ip-172-31-22-30:~$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-65f6dbf847-5snmw   1/1     Running   0          11m
kube-system   calico-node-5fl6m                          1/1     Running   0          11m
kube-system   calico-node-76dgp                          1/1     Running   0          7m23s
kube-system   calico-node-8wsn2                          1/1     Running   0          5m44s
kube-system   calico-node-lvrgp                          1/1     Running   0          6m25s
kube-system   calico-node-p5gs2                          1/1     Running   0          5m8s
kube-system   calico-node-zwdrl                          1/1     Running   0          5m26s
kube-system   coredns-66bc5c9577-chlxr                   1/1     Running   0          18m
kube-system   coredns-66bc5c9577-kr6dd                   1/1     Running   0          18m
kube-system   etcd-ip-172-31-17-28                       1/1     Running   0          6m25s
kube-system   etcd-ip-172-31-22-30                       1/1     Running   1          18m
kube-system   etcd-ip-172-31-27-126                      1/1     Running   0          7m22s
kube-system   kube-apiserver-ip-172-31-17-28             1/1     Running   0          6m25s
kube-system   kube-apiserver-ip-172-31-22-30             1/1     Running   1          18m
kube-system   kube-apiserver-ip-172-31-27-126            1/1     Running   0          7m22s
kube-system   kube-controller-manager-ip-172-31-17-28    1/1     Running   0          6m25s
kube-system   kube-controller-manager-ip-172-31-22-30    1/1     Running   1          18m
kube-system   kube-controller-manager-ip-172-31-27-126   1/1     Running   0          7m22s
kube-system   kube-proxy-6v7dm                           1/1     Running   0          5m8s
kube-system   kube-proxy-8lk6t                           1/1     Running   0          5m44s
kube-system   kube-proxy-j472s                           1/1     Running   0          5m26s
kube-system   kube-proxy-krgk7                           1/1     Running   0          7m23s
kube-system   kube-proxy-tw4fb                           1/1     Running   0          6m25s
kube-system   kube-proxy-tzckj                           1/1     Running   0          18m
kube-system   kube-scheduler-ip-172-31-17-28             1/1     Running   0          6m25s
kube-system   kube-scheduler-ip-172-31-22-30             1/1     Running   1          18m
kube-system   kube-scheduler-ip-172-31-27-126            1/1     Running   0          7m22s
ubuntu@ip-172-31-22-30:~$

```





# final output:

```bash
ubuntu@ip-172-31-26-102:~$ kubectl get nodes
NAME               STATUS   ROLES           AGE   VERSION
ip-172-31-18-245   Ready    control-plane   32m   v1.34.0
ip-172-31-20-143   Ready    <none>          26m   v1.34.0
ip-172-31-22-61    Ready    <none>          27m   v1.34.0
ip-172-31-26-102   Ready    control-plane   28m   v1.34.0
ip-172-31-26-94    Ready    <none>          26m   v1.34.0
ip-172-31-31-24    Ready    control-plane   27m   v1.34.0
ubuntu@ip-172-31-26-102:~$ kubectl get pods -A
NAMESPACE       NAME                                                              READY   STATUS      RESTARTS      AGE
demo            auth-svc-redis-master-0                                           1/1     Running     0             23m
demo            cl-mongodb-6d8dcf5548-2skfc                                       1/1     Running     0             23m
demo            kafka-controller-0                                                1/1     Running     0             23m
demo            moja-account-lookup-service-67b79c9b55-2tg8f                      1/1     Running     0             18m
demo            moja-account-lookup-service-admin-6578dcf5c-hhpqb                 1/1     Running     0             18m
demo            moja-als-msisdn-oracle-85d75f59d9-9f9zw                           1/1     Running     0             18m
demo            moja-centralledger-handler-admin-transfer-594d75bd87-j5p9f        1/1     Running     0             18m
demo            moja-centralledger-handler-timeout-7d64fbb54c-zz8ff               1/1     Running     0             18m
demo            moja-centralledger-handler-transfer-fulfil-7b845657db-xqbdq       1/1     Running     0             18m
demo            moja-centralledger-handler-transfer-get-775864df96-m5n7m          1/1     Running     0             18m
demo            moja-centralledger-handler-transfer-position-75db8844d7-5px8d     1/1     Running     0             18m
demo            moja-centralledger-handler-transfer-prepare-8c4644f69-clmlr       1/1     Running     0             18m
demo            moja-centralledger-service-fd944d8d8-mx7z2                        1/1     Running     0             18m
demo            moja-centralledger-service-migration-nq8zs                        0/1     Completed   0             19m
demo            moja-centralsettlement-handler-deferredsettlement-6f75cbf828mw8   1/1     Running     0             18m
demo            moja-centralsettlement-handler-rules-68c58bb56f-jmgwv             1/1     Running     0             18m
demo            moja-centralsettlement-service-6c4b8f8bcc-rn8pf                   1/1     Running     0             18m
demo            moja-handler-pos-batch-5f79f7578f-ml4x9                           1/1     Running     0             18m
demo            moja-ml-api-adapter-handler-notification-664f449786-n9wqn         1/1     Running     0             18m
demo            moja-ml-api-adapter-service-5658fcfc9d-p49t4                      1/1     Running     0             18m
demo            moja-ml-testing-toolkit-backend-0                                 1/1     Running     0             18m
demo            moja-ml-testing-toolkit-frontend-7dc99cd6df-lxmqf                 1/1     Running     0             18m
demo            moja-quoting-service-6f486754cc-jxt4c                             1/1     Running     0             18m
demo            moja-quoting-service-handler-7c4675c498-vvktm                     1/1     Running     0             18m
demo            moja-sim-payeefsp-backend-57f55979f5-f7frd                        1/1     Running     0             18m
demo            moja-sim-payeefsp-cache-5f949fc5cc-c2djz                          1/1     Running     0             18m
demo            moja-sim-payeefsp-scheme-adapter-86585b65c8-lq6np                 1/1     Running     1 (15m ago)   18m
demo            moja-sim-payerfsp-backend-cb7d6df5b-r9lx5                         1/1     Running     0             18m
demo            moja-sim-payerfsp-cache-66b8df8644-m49mb                          1/1     Running     0             18m
demo            moja-sim-payerfsp-scheme-adapter-66d548d7f7-9w9t4                 1/1     Running     1 (15m ago)   18m
demo            moja-sim-testfsp1-backend-65d667bfbb-l7bcq                        1/1     Running     0             18m
demo            moja-sim-testfsp1-cache-6d87cd45f8-fxcnv                          1/1     Running     0             18m
demo            moja-sim-testfsp1-scheme-adapter-78d766f6b7-98kch                 1/1     Running     1 (15m ago)   18m
demo            moja-sim-testfsp2-backend-547d499667-qgnsq                        1/1     Running     0             18m
demo            moja-sim-testfsp2-cache-66dcc8b7df-n8r9g                          1/1     Running     0             18m
demo            moja-sim-testfsp2-scheme-adapter-fb9d7f7d-nscdl                   1/1     Running     0             18m
demo            moja-sim-testfsp3-backend-76495dbbbd-5rx9f                        1/1     Running     0             18m
demo            moja-sim-testfsp3-cache-79cd797848-qkn9d                          1/1     Running     0             18m
demo            moja-sim-testfsp3-scheme-adapter-85588b6fd9-94klx                 1/1     Running     1 (15m ago)   18m
demo            moja-sim-testfsp4-backend-6b496d78d9-v49fq                        1/1     Running     0             18m
demo            moja-sim-testfsp4-cache-7d5f5fd64c-2wcp4                          1/1     Running     0             18m
demo            moja-sim-testfsp4-scheme-adapter-6ddd6cc79b-6whz9                 1/1     Running     0             18m
demo            moja-simulator-76db6b946-zv7sn                                    1/1     Running     0             18m
demo            moja-transaction-requests-service-5c8498687-zk2nn                 1/1     Running     0             18m
demo            mysqldb-0                                                         1/1     Running     0             23m
demo            proxy-cache-redis-0                                               1/1     Running     0             23m
demo            proxy-cache-redis-1                                               1/1     Running     0             23m
demo            proxy-cache-redis-2                                               1/1     Running     1 (22m ago)   23m
demo            proxy-cache-redis-3                                               1/1     Running     1 (22m ago)   23m
demo            proxy-cache-redis-4                                               1/1     Running     0             23m
demo            proxy-cache-redis-5                                               1/1     Running     0             23m
demo            ttk-mongodb-69c65f7688-fjrkw                                      1/1     Running     0             23m
demo            ttksims-redis-master-0                                            1/1     Running     0             23m
ingress-nginx   nginx-ingress-nginx-controller-7f4bcdbd44-m5m55                   1/1     Running     0             24m
kube-system     calico-kube-controllers-65f6dbf847-tr2nc                          1/1     Running     0             29m
kube-system     calico-node-62hq5                                                 1/1     Running     0             28m
kube-system     calico-node-g5hf5                                                 1/1     Running     0             26m
kube-system     calico-node-lnwp6                                                 1/1     Running     0             27m
kube-system     calico-node-sw5rn                                                 1/1     Running     0             26m
kube-system     calico-node-vfgcd                                                 1/1     Running     0             29m
kube-system     calico-node-wt2th                                                 1/1     Running     0             27m
kube-system     coredns-66bc5c9577-4xsbc                                          1/1     Running     0             32m
kube-system     coredns-66bc5c9577-k2b48                                          1/1     Running     0             32m
kube-system     etcd-ip-172-31-18-245                                             1/1     Running     0             32m
kube-system     etcd-ip-172-31-26-102                                             1/1     Running     0             28m
kube-system     etcd-ip-172-31-31-24                                              1/1     Running     0             27m
kube-system     kube-apiserver-ip-172-31-18-245                                   1/1     Running     0             32m
kube-system     kube-apiserver-ip-172-31-26-102                                   1/1     Running     0             28m
kube-system     kube-apiserver-ip-172-31-31-24                                    1/1     Running     0             27m
kube-system     kube-controller-manager-ip-172-31-18-245                          1/1     Running     0             32m
kube-system     kube-controller-manager-ip-172-31-26-102                          1/1     Running     0             28m
kube-system     kube-controller-manager-ip-172-31-31-24                           1/1     Running     0             27m
kube-system     kube-proxy-8xjlc                                                  1/1     Running     0             27m
kube-system     kube-proxy-b6nc6                                                  1/1     Running     0             26m
kube-system     kube-proxy-s9n5p                                                  1/1     Running     0             27m
kube-system     kube-proxy-w82xj                                                  1/1     Running     0             28m
kube-system     kube-proxy-wvw4z                                                  1/1     Running     0             32m
kube-system     kube-proxy-xskcl                                                  1/1     Running     0             26m
kube-system     kube-scheduler-ip-172-31-18-245                                   1/1     Running     0             32m
kube-system     kube-scheduler-ip-172-31-26-102                                   1/1     Running     0             28m
kube-system     kube-scheduler-ip-172-31-31-24                                    1/1     Running     0             27m
ubuntu@ip-172-31-26-102:~$ kubectl get pods -n demo
NAME                                                              READY   STATUS      RESTARTS      AGE
auth-svc-redis-master-0                                           1/1     Running     0             23m
cl-mongodb-6d8dcf5548-2skfc                                       1/1     Running     0             23m
kafka-controller-0                                                1/1     Running     0             23m
moja-account-lookup-service-67b79c9b55-2tg8f                      1/1     Running     0             18m
moja-account-lookup-service-admin-6578dcf5c-hhpqb                 1/1     Running     0             18m
moja-als-msisdn-oracle-85d75f59d9-9f9zw                           1/1     Running     0             18m
moja-centralledger-handler-admin-transfer-594d75bd87-j5p9f        1/1     Running     0             18m
moja-centralledger-handler-timeout-7d64fbb54c-zz8ff               1/1     Running     0             18m
moja-centralledger-handler-transfer-fulfil-7b845657db-xqbdq       1/1     Running     0             18m
moja-centralledger-handler-transfer-get-775864df96-m5n7m          1/1     Running     0             18m
moja-centralledger-handler-transfer-position-75db8844d7-5px8d     1/1     Running     0             18m
moja-centralledger-handler-transfer-prepare-8c4644f69-clmlr       1/1     Running     0             18m
moja-centralledger-service-fd944d8d8-mx7z2                        1/1     Running     0             18m
moja-centralledger-service-migration-nq8zs                        0/1     Completed   0             19m
moja-centralsettlement-handler-deferredsettlement-6f75cbf828mw8   1/1     Running     0             18m
moja-centralsettlement-handler-rules-68c58bb56f-jmgwv             1/1     Running     0             18m
moja-centralsettlement-service-6c4b8f8bcc-rn8pf                   1/1     Running     0             18m
moja-handler-pos-batch-5f79f7578f-ml4x9                           1/1     Running     0             18m
moja-ml-api-adapter-handler-notification-664f449786-n9wqn         1/1     Running     0             18m
moja-ml-api-adapter-service-5658fcfc9d-p49t4                      1/1     Running     0             18m
moja-ml-testing-toolkit-backend-0                                 1/1     Running     0             18m
moja-ml-testing-toolkit-frontend-7dc99cd6df-lxmqf                 1/1     Running     0             18m
moja-quoting-service-6f486754cc-jxt4c                             1/1     Running     0             18m
moja-quoting-service-handler-7c4675c498-vvktm                     1/1     Running     0             18m
moja-sim-payeefsp-backend-57f55979f5-f7frd                        1/1     Running     0             18m
moja-sim-payeefsp-cache-5f949fc5cc-c2djz                          1/1     Running     0             18m
moja-sim-payeefsp-scheme-adapter-86585b65c8-lq6np                 1/1     Running     1 (15m ago)   18m
moja-sim-payerfsp-backend-cb7d6df5b-r9lx5                         1/1     Running     0             18m
moja-sim-payerfsp-cache-66b8df8644-m49mb                          1/1     Running     0             18m
moja-sim-payerfsp-scheme-adapter-66d548d7f7-9w9t4                 1/1     Running     1 (15m ago)   18m
moja-sim-testfsp1-backend-65d667bfbb-l7bcq                        1/1     Running     0             18m
moja-sim-testfsp1-cache-6d87cd45f8-fxcnv                          1/1     Running     0             18m
moja-sim-testfsp1-scheme-adapter-78d766f6b7-98kch                 1/1     Running     1 (15m ago)   18m
moja-sim-testfsp2-backend-547d499667-qgnsq                        1/1     Running     0             18m
moja-sim-testfsp2-cache-66dcc8b7df-n8r9g                          1/1     Running     0             18m
moja-sim-testfsp2-scheme-adapter-fb9d7f7d-nscdl                   1/1     Running     0             18m
moja-sim-testfsp3-backend-76495dbbbd-5rx9f                        1/1     Running     0             18m
moja-sim-testfsp3-cache-79cd797848-qkn9d                          1/1     Running     0             18m
moja-sim-testfsp3-scheme-adapter-85588b6fd9-94klx                 1/1     Running     1 (15m ago)   18m
moja-sim-testfsp4-backend-6b496d78d9-v49fq                        1/1     Running     0             18m
moja-sim-testfsp4-cache-7d5f5fd64c-2wcp4                          1/1     Running     0             18m
moja-sim-testfsp4-scheme-adapter-6ddd6cc79b-6whz9                 1/1     Running     0             18m
moja-simulator-76db6b946-zv7sn                                    1/1     Running     0             18m
moja-transaction-requests-service-5c8498687-zk2nn                 1/1     Running     0             18m
mysqldb-0                                                         1/1     Running     0             23m
proxy-cache-redis-0                                               1/1     Running     0             23m
proxy-cache-redis-1                                               1/1     Running     0             23m
proxy-cache-redis-2                                               1/1     Running     1 (22m ago)   23m
proxy-cache-redis-3                                               1/1     Running     1 (22m ago)   23m
proxy-cache-redis-4                                               1/1     Running     0             23m
proxy-cache-redis-5                                               1/1     Running     0             23m
ttk-mongodb-69c65f7688-fjrkw                                      1/1     Running     0             23m
ttksims-redis-master-0                                            1/1     Running     0             23m
ubuntu@ip-172-31-26-102:~$

```


# testing output:

```bash
kubectl logs moja-ml-ttk-test-setup -n demo


--------------------FINAL REPORT--------------------

Test Suite:Standard Provisioning Collection
Environment:Development
┌───────────────────────────────────────────────────┐
│                      SUMMARY                      │
├───────────────────┬───────────────────────────────┤
│ Total assertions  │ 460                           │
├───────────────────┼───────────────────────────────┤
│ Passed assertions │ 457                           │
├───────────────────┼───────────────────────────────┤
│ Failed assertions │ 0                             │
├───────────────────┼───────────────────────────────┤
│ Total requests    │ 455                           │
├───────────────────┼───────────────────────────────┤
│ Total test cases  │ 49                            │
├───────────────────┼───────────────────────────────┤
│ Passed percentage │ 100.00%                        │
├───────────────────┼───────────────────────────────┤
│ Started time      │ Wed, 10 Sep 2025 10:43:12 GMT │
├───────────────────┼───────────────────────────────┤
│ Completed time    │ 2025-09-10T10:48:18.773Z      │
├───────────────────┼───────────────────────────────┤
│ Runtime duration  │ 305919 ms                     │
└───────────────────┴───────────────────────────────┘
/tmp/TTK-Assertion-Report-standard_provisioning_collection-2025-09-10T10:48:18.773Z.html was generated
Report saved on TTK backend server successfully and is available at testing-toolkit.local/api/history/test-reports/standard_provisioning_collection_2025-09-10T10:48:18.773Z?format=html

```
