

# HA Kubernetes Cluster Setup (Production-Grade)

This repository contains a complete step-by-step guide to deploy a **Highly Available Kubernetes Cluster** using `kubeadm`, `HAProxy` load balancer, and the `Calico` CNI plugin.

It is designed to replicate **real-world production environments**, following industry best practices for **resiliency, fault tolerance, and scalability**.

---

## 📖 About this Documentation

This documentation has been authored by **Nahidul Islam**, based on real-world, production-grade deployment experience as a **DevOps & Cloud Engineer**.
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

#---

ubuntu@ip-172-31-18-245:~$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-65f6dbf847-tr2nc   1/1     Running   0          62m
calico-node-62hq5                          1/1     Running   0          61m
calico-node-g5hf5                          1/1     Running   0          59m
calico-node-lnwp6                          1/1     Running   0          60m
calico-node-sw5rn                          1/1     Running   0          59m
calico-node-vfgcd                          1/1     Running   0          62m
calico-node-wt2th                          1/1     Running   0          60m
coredns-66bc5c9577-4xsbc                   1/1     Running   0          65m
coredns-66bc5c9577-k2b48                   1/1     Running   0          65m
etcd-ip-172-31-18-245                      1/1     Running   0          65m
etcd-ip-172-31-26-102                      1/1     Running   0          61m
etcd-ip-172-31-31-24                       1/1     Running   0          60m
kube-apiserver-ip-172-31-18-245            1/1     Running   0          65m
kube-apiserver-ip-172-31-26-102            1/1     Running   0          61m
kube-apiserver-ip-172-31-31-24             1/1     Running   0          60m
kube-controller-manager-ip-172-31-18-245   1/1     Running   0          65m
kube-controller-manager-ip-172-31-26-102   1/1     Running   0          61m
kube-controller-manager-ip-172-31-31-24    1/1     Running   0          60m
kube-proxy-8xjlc                           1/1     Running   0          60m
kube-proxy-b6nc6                           1/1     Running   0          59m
kube-proxy-s9n5p                           1/1     Running   0          60m
kube-proxy-w82xj                           1/1     Running   0          61m
kube-proxy-wvw4z                           1/1     Running   0          65m
kube-proxy-xskcl                           1/1     Running   0          59m
kube-scheduler-ip-172-31-18-245            1/1     Running   0          65m
kube-scheduler-ip-172-31-26-102            1/1     Running   0          61m
kube-scheduler-ip-172-31-31-24             1/1     Running   0          60m

#---

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




# Golden Path test collection log output summery


```bash

ubuntu@ip-172-31-18-245:~$ kubectl -n demo logs pod/moja-ml-ttk-test-setup
Server:		10.96.0.10
Address:	10.96.0.10:53

Non-authoritative answer:
Name:	github.com
Address: 140.82.112.4

Non-authoritative answer:

Downloading the test collection...
Connecting to github.com (140.82.112.4:443)
Connecting to codeload.github.com (140.82.112.9:443)
saving to '/tmp/downloaded-test-collections.zip'
downloaded-test-coll 100% |********************************|  862k  0:00:00 ETA
'/tmp/downloaded-test-collections.zip' saved
Archive:  /tmp/downloaded-test-collections.zip
   creating: testing-toolkit-test-cases-17.1.0/
  inflating: testing-toolkit-test-cases-17.1.0/.gitignore
  inflating: testing-toolkit-test-cases-17.1.0/LICENSE.md
  inflating: testing-toolkit-test-cases-17.1.0/README.md
   creating: testing-toolkit-test-cases-17.1.0/collections/
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/additional-tests/
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/additional-tests/p2p_scheme_adapter_websocket.json
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/get_quotes_transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/negative_scenarios/parties_negative.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/negative_scenarios/quotes_negative.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/negative_scenarios/transfers_negative.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/golden_path/p2p_happy_path.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/provision_mojaloop_simulator.json
   creating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/proxy_testing/
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/proxy_testing/proxy1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/proxy_testing/proxy2.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/dfsp/provisioning/proxy_testing/testdfsp.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/Active_Inactive_participants/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/Active_Inactive_participants/active_and_inactive_participant.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/Active_Inactive_participants/active_and_inactive_participants_accounts.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/Active_Inactive_participants/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/backward_compatibility/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/backward_compatibility/fspiop_protocol_validation.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/block_transfer.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/transfers/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/transfers/fulfill_commit.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/transfers/fulfill_reject.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/transfers/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/duplicate_handling/transfers/original_transfer_at_committed.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_in/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_in/funds_in_ttk.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/Reserve&Abort/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/Reserve&Abort/funds_out_abort.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/Reserve&Commit/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/Reserve&Commit/funds_out_commit.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/funds_out/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/get_transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/p2p_money_transfer/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/p2p_money_transfer/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_subid.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/patch_notifications/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/patch_notifications/patch_notification.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/fulfil-reserved-v1.0.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/payee_abort_v1.1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/payee_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/payee_invalid_timestamp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/feature_tests/transfer_negative_scenarios/payer_transfer_timeout.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/api_schema_validation/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/api_schema_validation/fx_quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/api_schema_validation/fx_transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/api_schema_validation/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/duplicate_handling/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/duplicate_handling/duplicate_fx_transfers.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/happy_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/happy_path/fx_tests.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/fxp_error.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/fxp_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/payee_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/feature_tests/negative_scenarios/timeout_scenarios.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/fx/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/Error-framework-authorizations.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/authorizations.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/error-framework.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/received State.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/rejected State.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/HA/golden_path/transaction_request_service/transaction_request_service_health.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/cleanup/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/cleanup/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/cleanup/position_cleanup.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/cleanup/position_cleanup_prod.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/SettlementWindows/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/SettlementWindows/settlementadmin.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/admin-api-tests/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/admin-api-tests/Admintests-20201221.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/api-tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Test for 4 decimal points #949.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Test for Bugfix #1378 - extension list missing.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Test for Bugfix #2697 - Fulfil Handler does not correctly invalidate requests.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Test for Bugfix #742 - Error code check.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Test for Bugfix #849 - missing ID for transfers and quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Tests for Bugfix #1009 - ML Adapter and ALS service health should include broker status.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Tests for Bugfix #981 - Fix 500 http code instead of 400.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Tests for Bugfix #990 and #1016 - Quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/Tests for Bugfix #998 - Quoting service not using most recent endpoint.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/bug fixes/other-bug-fixes.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/Active_Inactive_participants/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/Active_Inactive_participants/active_and_inactive_participant.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/Active_Inactive_participants/active_and_inactive_participants_accounts.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/Active_Inactive_participants/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/backward_compatibility/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/backward_compatibility/fspiop_protocol_validation.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/block_transfer.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/transfers/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/transfers/fulfill_commit.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/transfers/fulfill_reject.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/transfers/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/duplicate_handling/transfers/original_transfer_at_committed.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_in/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_in/funds_in_ttk.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/Reserve&Abort/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/Reserve&Abort/funds_out_abort.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/Reserve&Commit/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/Reserve&Commit/funds_out_commit.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/funds_out/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/get_transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_accent_unicode.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_burmese_unicode.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_subid.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_subid_error_callback.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_with_balance_checks.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/p2p_money_transfer/p2p_happy_path_with_balance_checks_ttk.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/patch_notifications/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/patch_notifications/patch_notification.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/post_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/post_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/post_scenarios/positive.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/post_scenarios/reserve_notification_positive_testfsp1_testfsp2.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/resources_based/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/resources_based/participants/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/resources_based/participants/participants.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/fulfil-reserved-v1.0.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/payee_abort_v1.1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/payee_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/payee_invalid_timestamp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/feature_tests/transfer_negative_scenarios/payer_transfer_timeout.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/api_schema_validation/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/api_schema_validation/fx_quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/api_schema_validation/fx_transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/api_schema_validation/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/duplicate_handling/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/duplicate_handling/duplicate_fx_transfers.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/happy_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/happy_path/fx_tests.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/fxp_error.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/fxp_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/payee_invalid_fulfillment.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/feature_tests/negative_scenarios/timeout_scenarios.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/fx/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/negative_scenarios/quotes_negative.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/p2p_on_us_transfers/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/p2p_on_us_transfers/p2p_money_transfer_on_us.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/quoting_service/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/quoting_service/quoting_service.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management/Settlement-management-primary-currency-test.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management/Settlement-management-second-currency-test.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management/mixed_settlement_model.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management_prod/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/settlement_management_prod/mixed_settlement_model.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/Error-framework-authorizations.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/authorizations.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/error-framework.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/received State.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/rejected State.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path/transaction_request_service/transaction_request_service_health.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path_fspiop/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path_fspiop/api-tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path_fspiop/api-tests/Quotes/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/golden_path_fspiop/api-tests/Quotes/quotes-negative-scenarios.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/happy_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/happy_path/discovery.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/happy_path/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/happy_path/quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/happy_path/transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/negative_scenarios/discovery.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/negative_scenarios/quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payee_scheme/negative_scenarios/transfers.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/happy_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/happy_path/discovery.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/happy_path/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/happy_path/quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/happy_path/transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/negative_scenarios/discovery.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/negative_scenarios/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/negative_scenarios/quotes.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/as_payer_scheme/negative_scenarios/transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/inter_scheme/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/bulk-duplicated-transfers.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/bulk-invalid-dfsp-name-header-and-body.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/bulk-invalid-timestamp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/bulk-transfer-timeout-scenario.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/bulk-warm-up.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/fspiop_protocol_validation.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/negative_scenarios.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/bulk_transfers/positive_scenarios.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/merchant/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/merchant/merchant-lookup.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/new_changes/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/new_changes/get_parties_subid.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/settlement_cgs/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/settlement_cgs/newsetcgs.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/settlement_fx/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/settlement_fx/settlement_tests.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/v1.1_changes/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/other_tests/v1.1_changes/patch_notification.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/CGS_Specific/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/CGS_Specific/add-users-to-new-sims-and-als-registration.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/CGS_Specific/adjust-participants-limits.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/CGS_Specific/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/CGS_Specific/oracle-onboarding.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopHub_Setup/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopHub_Setup/hub.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopHub_Setup_prod/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopHub_Setup_prod/hub.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/noresponsepayeefsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/payeefsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/payerfsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/testfsp1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/testfsp2.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/testfsp3.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/testfsp4.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/testingtoolkitdfsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/ttkfxp1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/ttkfxpayee.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/ttkfxpayer.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/MojaloopSims_Onboarding/ttkpayeefsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_inter_scheme/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_inter_scheme/new_proxy.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_inter_scheme/proxy_2.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_sdk_bulk/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_sdk_bulk/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_sdk_bulk/ttksim1.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_sdk_bulk/ttksim2.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_sdk_bulk/ttksim3.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/centralauth.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/dfspa.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/dfspb.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/dfspb_parties.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/hub.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_thirdparty/pisp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_hub/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_hub/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_hub/new_hub.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_participants/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_participants/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_participants/new_dfsp.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/new_participants/new_fxp.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning_merchant/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning_merchant/README.md
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/provisioning_merchant/merchant_setup.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/bulk-happy-path-mvp-out-of-scope.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/bulk-quotes-error-cases-mvp-out-of-scope.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/bulk-transfers-error-case-mvp-out-of-scope.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/legacy-high-volume-bulk.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk-mvp-out-of-scope/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-happy-path.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-high-volume_dynamic.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-parties-error-cases.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-quotes-error-cases.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-sdk-warm-up.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/bulk-transfers-error-cases.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/bulk/basic/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/request-to-pay/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/request-to-pay/basic/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sdk_scheme_adapter/request-to-pay/basic/happy-path.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/Create Oracle Endpoints - [seq-acct-lookup-admin-post-oracle-7.3.2].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/Delete Oracle Endpoints - [seq-acct-lookup-admin-delete-oracle-7.3.4].json.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/Get Oracles - [seq-acct-lookup-admin-get-oracle-7.3.1].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/Update Oracle Endpoints - [seq-acct-lookup-admin-put-oracle-7.3.3].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service-admin/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/Create Batch Participants - [seq-acct-lookup-post-participants-batch-7.1.1].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/Create Participant - [seq-acct-lookup-post-participant-7.1.3].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/Delete Participant Details - [seq-acct-lookup-del-participants-7.1.2].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/Get Participant Details - [seq-acct-lookup-get-participants-7.1.0].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/Get Party Details - [seq-acct-lookup-get-parties-7.2.0].json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/account-lookup-service/master.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/master.json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/quoting-service/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/sequence/quoting-service/Quoting Service Sequences [seq-quote-1.0.0].json
   creating: testing-toolkit-test-cases-17.1.0/collections/hub/thirdparty/
  inflating: testing-toolkit-test-cases-17.1.0/collections/hub/thirdparty/collection.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/happy_path/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/happy_path/sdk_fx_transfer.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/negative_scenarios/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/negative_scenarios/fxp_error.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/negative_scenarios/fxp_non_success_states.json
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/forex/feature_tests/negative_scenarios/fxp_timeout_scenarios.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/p2p/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/p2p/sdk_post_transfer.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/proxy/
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/proxy/fxp/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/golden_path/proxy/fxp/sdk_post_transfer.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/provisioning/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/provisioning/sdk_post_accounts.json
   creating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/provisioning_fxp/
  inflating: testing-toolkit-test-cases-17.1.0/collections/pm4ml/provisioning_fxp/from-switch.json
   creating: testing-toolkit-test-cases-17.1.0/definitions/
  inflating: testing-toolkit-test-cases-17.1.0/definitions/TTK-Testcase-Definition-GP-v13.0.1.html
   creating: testing-toolkit-test-cases-17.1.0/environments/
  inflating: testing-toolkit-test-cases-17.1.0/environments/hub.json
   creating: testing-toolkit-test-cases-17.1.0/environments/hub/
   creating: testing-toolkit-test-cases-17.1.0/environments/hub/forex/
  inflating: testing-toolkit-test-cases-17.1.0/environments/hub/forex/fxp-env.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/hub_cgs.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/pm4ml-env.json
   creating: testing-toolkit-test-cases-17.1.0/environments/pm4ml/
   creating: testing-toolkit-test-cases-17.1.0/environments/pm4ml/proxy/
   creating: testing-toolkit-test-cases-17.1.0/environments/pm4ml/proxy/fxp/
  inflating: testing-toolkit-test-cases-17.1.0/environments/pm4ml/proxy/fxp/proxy-fxp-env.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/pm4ml_fxp_provisioning_from_switch.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/provisioning_dfsp.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/provisioning_merchant.json
  inflating: testing-toolkit-test-cases-17.1.0/environments/sdk-bulk.json
  inflating: testing-toolkit-test-cases-17.1.0/package.json
   creating: testing-toolkit-test-cases-17.1.0/rules/
   creating: testing-toolkit-test-cases-17.1.0/rules/hub/
   creating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_callback/
  inflating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_callback/default.json
   creating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_response/
  inflating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_response/default.json
   creating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_validation/
  inflating: testing-toolkit-test-cases-17.1.0/rules/hub/rules_validation/default.json
   creating: testing-toolkit-test-cases-17.1.0/rules/mojaloop/
   creating: testing-toolkit-test-cases-17.1.0/rules/mojaloop/ml-testing-toolkit/
   creating: testing-toolkit-test-cases-17.1.0/rules/mojaloop/ml-testing-toolkit/spec_files/
   creating: testing-toolkit-test-cases-17.1.0/rules/mojaloop/ml-testing-toolkit/spec_files/rules_callback/
  inflating: testing-toolkit-test-cases-17.1.0/rules/mojaloop/ml-testing-toolkit/spec_files/rules_callback/default_iso.json
   creating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/
   creating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/forex/
   creating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/forex/rules_sync_response/
  inflating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/forex/rules_sync_response/default.json
  inflating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/forex/rules_sync_response/fxp_response_rules.json
   creating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/proxy/
   creating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/proxy/rules_sync_response/
  inflating: testing-toolkit-test-cases-17.1.0/rules/pm4ml/proxy/rules_sync_response/default.json
   creating: testing-toolkit-test-cases-17.1.0/rules/sdk-bulk/
  inflating: testing-toolkit-test-cases-17.1.0/rules/sdk-bulk/README.md
   creating: testing-toolkit-test-cases-17.1.0/scripts/
  inflating: testing-toolkit-test-cases-17.1.0/scripts/env-compare.js
  inflating: testing-toolkit-test-cases-17.1.0/scripts/set-accept-content-type-headers.js

> @mojaloop/ml-testing-toolkit-client-lib@1.10.3 cli
> node src/client.js -c cli-default-config.json -e cli-testcase-environment.json -i /tmp/test_cases/testing-toolkit-test-cases-17.1.0/collections/hub/provisioning/for_golden_path -u http://moja-ml-testing-toolkit-backend:5050 --report-format html --report-auto-filename-enable true --save-report true --report-name standard_provisioning_collection --report-folder /tmp --save-report-base-url testing-toolkit.local --extra-summary-information=Test Suite:Standard Provisioning Collection,Environment:Development

Listening on http://moja-ml-testing-toolkit-backend:5050 outboundProgress events...
(node:19) NOTE: The AWS SDK for JavaScript (v2) is in maintenance mode.
 SDK releases are limited to address critical bug fixes and security issues only.

Please migrate your code to use AWS SDK for JavaScript (v3).
For more information, check the blog post at https://a.co/cUPnyil
(Use `node --trace-warnings ...` to show where the warning was created)
[ 1 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 2 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 3 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Hub Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 4 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Hub Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 5 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Hub Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 6 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Second Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 7 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION Second Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 8 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 9 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 10 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Third Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 11 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION Third Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 12 passed, 0 skipped, 0 failed of 460 ]
  Settlement Models -> Create settlement model DEFERRED NET XXX
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 13 passed, 0 skipped, 0 failed of 460 ]
  Settlement Models -> Create settlement model DEFAULT DEFERRED NET
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 14 passed, 0 skipped, 0 failed of 460 ]
  Settlement Models -> Create settlement model Continous Gross
	[ SUCCESS ]	settlement model created code 201
[ 15 passed, 0 skipped, 0 failed of 460 ]
  Settlement Models -> Create settlement model Interchange Fee
	[ SUCCESS ]	settlement model created code 201
[ 16 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding -> Register MSISDN Oracle
	[ SUCCESS ]	status to be 201 or errorCode 2001 already exists
[ 17 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding -> Register BUSINESS Oracle
	[ SUCCESS ]	status to be 201 or errorCode 2001 already exists
[ 18 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding -> Register ALIAS Oracle
	[ SUCCESS ]	status to be 201 or errorCode 2001 already exists
[ 19 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 20 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 21 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Second Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 22 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION Second Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 23 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Third Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 24 passed, 0 skipped, 0 failed of 460 ]
  Hub Account -> Add Hub Account-HUB_RECONCILIATION Third Currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 25 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add payerfsp
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 26 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add initial position and limits - payerfsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 27 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> payerfsp Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 28 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Deposit Funds in Settlement Account - payerfsp
	[ SUCCESS ]	status to be 202
[ 30 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> payerfsp Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	payerfsp Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 31 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add payeefsp with different currency for default settlement case
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 32 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add initial position and limits with different currency - payerfsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 33 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add payeefsp with different currency for default settlement case CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 34 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Add initial position and limits with different currency - payerfsp CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 35 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 36 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 38 passed, 0 skipped, 0 failed of 460 ]
  payerfsp account -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 39 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - AUTHORIZATIONS
	[ SUCCESS ]	Status code is 201
[ 40 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 41 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 42 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 43 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 44 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 45 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 46 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES PUT Error
	[ SUCCESS ]	Status code is 201
[ 47 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - QUOTES
	[ SUCCESS ]	Status code is 201
[ 48 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - TXN REQUEST
	[ SUCCESS ]	Status code is 201
[ 49 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 50 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 51 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 52 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - BULK-TRANSFER POST
	[ SUCCESS ]	Status code is 201
[ 53 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - BULK-TRANSFER PUT
	[ SUCCESS ]	Status code is 201
[ 54 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - BULK-TRANSFER ERROR
	[ SUCCESS ]	Status code is 201
[ 55 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 56 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 57 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 58 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 59 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 60 passed, 0 skipped, 0 failed of 460 ]
  payerfsp callbacks -> Add payerfsp callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 61 passed, 0 skipped, 0 failed of 460 ]
  payerfsp notification_emails -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 62 passed, 0 skipped, 0 failed of 460 ]
  payerfsp notification_emails -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 63 passed, 0 skipped, 0 failed of 460 ]
  payerfsp notification_emails -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 64 passed, 0 skipped, 0 failed of 460 ]
  payerfsp add_users -> Add SubId User - payerIdType - payerIdentifier - 100
	[ SUCCESS ]	Status code is 202
[ 65 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add payeefsp
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 66 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add initial position and limits - payeefsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 67 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> payeefsp Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 68 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Deposit Funds in Settlement Account - payeefsp
	[ SUCCESS ]	status to be 202
[ 70 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> payeefsp Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	payeefsp Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 71 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add payeefsp with different currency for default settlement case
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 72 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add initial position and limits with different currency - payeefsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 73 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add payeefsp with different currency for default settlement case CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 74 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Add initial position and limits with different currency - payeefsp CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 75 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 76 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 78 passed, 0 skipped, 0 failed of 460 ]
  payeefsp account -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 79 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - AUTHORIZATIONS
	[ SUCCESS ]	Status code is 201
[ 80 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 81 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 82 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 83 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 84 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 85 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 86 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - PARTIES PUT Error
	[ SUCCESS ]	Status code is 201
[ 87 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - QUOTES
	[ SUCCESS ]	Status code is 201
[ 88 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - TXN REQUEST
	[ SUCCESS ]	Status code is 201
[ 89 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 90 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 91 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 92 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - BULK-TRANSFER POST
	[ SUCCESS ]	Status code is 201
[ 93 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - BULK-TRANSFER PUT
	[ SUCCESS ]	Status code is 201
[ 94 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add payeefsp callback - BULK-TRANSFER ERROR
	[ SUCCESS ]	Status code is 201
[ 95 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 96 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 97 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 98 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 99 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 100 passed, 0 skipped, 0 failed of 460 ]
  payeefsp callbacks -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 101 passed, 0 skipped, 0 failed of 460 ]
  payeefsp notification_emails -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 102 passed, 0 skipped, 0 failed of 460 ]
  payeefsp notification_emails -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 103 passed, 0 skipped, 0 failed of 460 ]
  payeefsp notification_emails -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 104 passed, 0 skipped, 0 failed of 460 ]
  payeefsp oracle_registration -> Add payeeIdentifier in payeeIdType Oracle
	[ SUCCESS ]	Status code is 202
[ 105 passed, 0 skipped, 0 failed of 460 ]
  payeefsp add_users -> Add SubId User - payeeIdType - payeeIdentifier - 100
	[ SUCCESS ]	Status code is 202
[ 106 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add noresponsepayeefsp
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 107 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add initial position and limits - noresponsepayeefsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 108 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> noresponsepayeefsp Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 109 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Deposit Funds in Settlement Account - noresponsepayeefsp
	[ SUCCESS ]	status to be 202
[ 111 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> noresponsepayeefsp Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	noresponsepayeefsp Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 112 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add noresponsepayeefsp with different currency for default settlement case
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 113 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add initial position and limits with different currency - noresponsepayeefsp
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 114 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add noresponsepayeefsp with different currency for default settlement case CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 115 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Add initial position and limits with different currency - noresponsepayeefsp CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 116 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 117 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 119 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp account -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 120 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - AUTHORIZATIONS
	[ SUCCESS ]	Status code is 201
[ 121 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 122 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 123 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 124 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 125 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 126 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 127 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - PARTIES PUT Error
	[ SUCCESS ]	Status code is 201
[ 128 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - QUOTES
	[ SUCCESS ]	Status code is 201
[ 129 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - TXN REQUEST
	[ SUCCESS ]	Status code is 201
[ 130 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 131 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 132 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 133 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - BULK-TRANSFER POST
	[ SUCCESS ]	Status code is 201
[ 134 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - BULK-TRANSFER PUT
	[ SUCCESS ]	Status code is 201
[ 135 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add noresponsepayeefsp callback - BULK-TRANSFER ERROR
	[ SUCCESS ]	Status code is 201
[ 136 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 137 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 138 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 139 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 140 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 141 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp callbacks -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 142 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp notification_emails -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 143 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp notification_emails -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 144 passed, 0 skipped, 0 failed of 460 ]
  noresponsepayeefsp notification_emails -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 145 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add testfsp1
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 146 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add initial position and limits - testfsp1
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 147 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> testfsp1 Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 148 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Deposit Funds in Settlement Account - testfsp1
	[ SUCCESS ]	status to be 202
[ 150 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> testfsp1 Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	testfsp1 Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 151 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add testfsp1 with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 152 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add initial position and limits - testfsp1 with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 153 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add testfsp1 with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 154 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Add initial position and limits - testfsp1 with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 155 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 156 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 158 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 account -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 159 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 160 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 161 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES PUT Error
	[ SUCCESS ]	Status code is 201
[ 162 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 163 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 164 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 165 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 166 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - QUOTES
	[ SUCCESS ]	Status code is 201
[ 167 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 168 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 169 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 170 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 171 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 172 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 173 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 174 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 175 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 callbacks -> Add testfsp1 callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 176 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 notification_emails -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 177 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 notification_emails -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 178 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 notification_emails -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 179 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 oracle_registration -> Add payeeIdentifier in payeeIdType Oracle
	[ SUCCESS ]	Status code is 202
[ 180 passed, 0 skipped, 0 failed of 460 ]
  testfsp1 add_users -> Add SubId User - testfsp1IdType - testfsp1Identifier - 100
	[ SUCCESS ]	Status code is 202
[ 181 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add testfsp2
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 182 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add initial position and limits - testfsp2
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 183 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> testfsp2 Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 184 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Deposit Funds in Settlement Account - testfsp2
	[ SUCCESS ]	status to be 202
[ 186 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> testfsp2 Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	testfsp2 Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 187 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add testfsp2 with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 188 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add initial position and limits - testfsp2 with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 189 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add testfsp2 with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 190 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Add initial position and limits - testfsp2 with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 191 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 192 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 194 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 account -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 195 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 196 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 197 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES PUT Error
	[ SUCCESS ]	Status code is 201
[ 198 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 199 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 200 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 201 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 202 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - QUOTES
	[ SUCCESS ]	Status code is 201
[ 203 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 204 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 205 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 206 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 207 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 208 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 209 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 210 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 211 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 callbacks -> Add testfsp2 callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 212 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 notification_emails -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 213 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 notification_emails -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 214 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 notification_emails -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 215 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 oracle_registration -> Add payeeIdentifier in payeeIdType Oracle
	[ SUCCESS ]	Status code is 202
[ 216 passed, 0 skipped, 0 failed of 460 ]
  testfsp2 add_users -> Add SubId User - testfsp2IdType - testfsp2Identifier - 100
	[ SUCCESS ]	Status code is 202
[ 217 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add participant
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 218 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add initial position and limits
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 219 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add participant with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 220 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add initial position and limits with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 221 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add participant with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 222 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add initial position and limits with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 223 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 224 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 225 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 226 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 227 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - QUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 228 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 229 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 230 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 231 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 232 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 233 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 234 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 235 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 236 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 237 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE
	[ SUCCESS ]	Status code is 201
[ 238 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 239 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 240 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 241 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk POST endpoint
	[ SUCCESS ]	Status code is 201
[ 242 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT endpoint
	[ SUCCESS ]	Status code is 201
[ 243 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT /error endpoint
	[ SUCCESS ]	Status code is 201
[ 244 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add testingtoolkitdfsp callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 245 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add testingtoolkitdfsp callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 246 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT ERROR
	[ SUCCESS ]	Status code is 201
[ 247 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 248 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 249 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 250 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FXQUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 251 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> testingtoolkitdfsp Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 252 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> Deposit Funds in Settlement Account - testingtoolkitdfsp
	[ SUCCESS ]	status to be 202
[ 254 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> testingtoolkitdfsp Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	testingtoolkitdfsp Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 255 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 256 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 258 passed, 0 skipped, 0 failed of 460 ]
  testingtoolkitdfsp fundsin -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 259 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add participant
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 260 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add initial position and limits
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 261 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 262 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 263 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 264 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 265 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - QUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 266 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 267 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 268 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 269 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 270 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 271 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 272 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 273 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 274 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 275 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE
	[ SUCCESS ]	Status code is 201
[ 276 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 277 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 278 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 279 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Setup Bulk POST endpoint
	[ SUCCESS ]	Status code is 201
[ 280 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Setup Bulk PUT endpoint
	[ SUCCESS ]	Status code is 201
[ 281 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Setup Bulk PUT /error endpoint
	[ SUCCESS ]	Status code is 201
[ 282 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add payerfsp callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 283 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add payerfsp callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 284 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add callback - PARTIES PUT ERROR
	[ SUCCESS ]	Status code is 201
[ 285 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add participant with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 286 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add initial position and limits with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 287 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add participant with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 288 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add initial position and limits with second currency CGS
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 289 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add payeeIdentifier in payeeIdType Oracle
	[ SUCCESS ]	Status code is 202
[ 290 passed, 0 skipped, 0 failed of 460 ]
  ttkpayeefsp provisioning -> Add payeeIdentifier in payeeIdType Oracle
	[ SUCCESS ]	Status code is 202
[ 291 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3 -> Add testfsp3
	[ SUCCESS ]	response code 201
[ 292 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3 -> [NBC=0]Add initial position and limits - testfsp3
	[ SUCCESS ]	response code is 201
[ 293 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3 -> GET /participants testfsp3
	[ SUCCESS ]	resposne code is 200
[ 294 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3 -> Deposit Funds in Settlement Account - testfsp3
	[ SUCCESS ]	response code is 202
[ 295 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - AUTHORIZATIONS
	[ SUCCESS ]	resposne code 201
[ 296 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTICIPANT PUT
	[ SUCCESS ]	resposne code 201
[ 297 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTICIPANT PUT Error
	[ SUCCESS ]	resposne code 201
[ 298 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	resposne code 201
[ 299 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	resposne code 201
[ 300 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTIES GET
	[ SUCCESS ]	resposne code 201
[ 301 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTIES PUT
	[ SUCCESS ]	resposne code 201
[ 302 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - PARTIES PUT Error
	[ SUCCESS ]	resposne code 201
[ 303 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - QUOTES
	[ SUCCESS ]	resposne code 201
[ 304 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - TXN REQUEST
	[ SUCCESS ]	resposne code 201
[ 305 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - TRANSFER POST
	[ SUCCESS ]	resposne code 201
[ 306 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - TRANSFER PUT
	[ SUCCESS ]	resposne code 201
[ 307 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks -> Add testfsp3 callback - TRANSFER ERROR
	[ SUCCESS ]	resposne code 201
[ 307 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-notifications -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
[ 307 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-notifications -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
[ 307 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-notifications -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
[ 308 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4 -> Add testfsp4
	[ SUCCESS ]	response code 201
[ 309 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4 -> [NBC=0]Add initial position and limits - testfsp4
	[ SUCCESS ]	response code is 201
[ 310 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4 -> GET /participants testfsp4
	[ SUCCESS ]	resposne code is 200
[ 311 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4 -> Deposit Funds in Settlement Account - testfsp4
	[ SUCCESS ]	response code is 202
[ 312 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - AUTHORIZATIONS
	[ SUCCESS ]	resposne code 201
[ 313 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTICIPANT PUT
	[ SUCCESS ]	resposne code 201
[ 314 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTICIPANT PUT Error
	[ SUCCESS ]	resposne code 201
[ 315 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	resposne code 201
[ 316 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	resposne code 201
[ 317 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTIES GET
	[ SUCCESS ]	resposne code 201
[ 318 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTIES PUT
	[ SUCCESS ]	resposne code 201
[ 319 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - PARTIES PUT Error
	[ SUCCESS ]	resposne code 201
[ 320 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - QUOTES
	[ SUCCESS ]	resposne code 201
[ 321 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - TXN REQUEST
	[ SUCCESS ]	resposne code 201
[ 322 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - TRANSFER POST
	[ SUCCESS ]	resposne code 201
[ 323 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - TRANSFER PUT
	[ SUCCESS ]	resposne code 201
[ 324 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks -> Add testfsp4 callback - TRANSFER ERROR
	[ SUCCESS ]	resposne code 201
[ 324 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-notifications -> Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
[ 324 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-notifications -> Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
[ 324 passed, 0 skipped, 0 failed of 460 ]
  FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-notifications -> Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
[ 325 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add participant
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 326 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add initial position and limits
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 327 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add participant with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 328 passed, 0 skipped, 0 failed of 460 ]
  Additional Currencies -> Add initial position and limits with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 329 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 330 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 331 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 332 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 333 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - QUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 334 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 335 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 336 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 337 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 338 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 339 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 340 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 341 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 342 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 343 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE
	[ SUCCESS ]	Status code is 201
[ 344 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 345 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 346 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 347 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk POST endpoint
	[ SUCCESS ]	Status code is 201
[ 348 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT endpoint
	[ SUCCESS ]	Status code is 201
[ 349 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT /error endpoint
	[ SUCCESS ]	Status code is 201
[ 350 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxp1 callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 351 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxp1 callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 352 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT ERROR
	[ SUCCESS ]	Status code is 201
[ 353 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 354 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 355 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 356 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FXQUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 357 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Get Status Request before deposit for currency2
	[ SUCCESS ]	status to be 200
[ 358 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Deposit Funds in Settlement Account currency2
	[ SUCCESS ]	status to be 202
[ 360 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Get Status Request after deposit for currency2
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 361 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add participant with second currency
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 362 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add initial position and limits with second currency
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 363 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 364 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 365 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 366 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 367 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - QUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 368 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 369 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 370 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 371 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 372 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 373 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 374 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 375 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 376 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 377 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE
	[ SUCCESS ]	Status code is 201
[ 378 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 379 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 380 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 381 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk POST endpoint
	[ SUCCESS ]	Status code is 201
[ 382 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT endpoint
	[ SUCCESS ]	Status code is 201
[ 383 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT /error endpoint
	[ SUCCESS ]	Status code is 201
[ 384 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxpayee callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 385 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxpayee callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 386 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT ERROR
	[ SUCCESS ]	Status code is 201
[ 387 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 388 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 389 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 390 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FXQUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 391 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add participant
	[ SUCCESS ]	status to be 201 if not exists or 400 if exists
[ 392 passed, 0 skipped, 0 failed of 460 ]
  Primary Account -> Add initial position and limits
	[ SUCCESS ]	status to be 201 if not exists or 500 if exists
[ 393 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT
	[ SUCCESS ]	Status code is 201
[ 394 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT PUT Error
	[ SUCCESS ]	Status code is 201
[ 395 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES GET
	[ SUCCESS ]	Status code is 201
[ 396 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT
	[ SUCCESS ]	Status code is 201
[ 397 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - QUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 398 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 399 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 400 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 401 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 402 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID PUT Error
	[ SUCCESS ]	Status code is 201
[ 403 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTICIPANT SUB-ID DELETE
	[ SUCCESS ]	Status code is 201
[ 404 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID GET
	[ SUCCESS ]	Status code is 201
[ 405 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID PUT
	[ SUCCESS ]	Status code is 201
[ 406 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES SUB-ID ERROR PUT
	[ SUCCESS ]	Status code is 201
[ 407 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE
	[ SUCCESS ]	Status code is 201
[ 408 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL
	[ SUCCESS ]	Status code is 201
[ 409 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL
	[ SUCCESS ]	Status code is 201
[ 410 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL
	[ SUCCESS ]	Status code is 201
[ 411 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk POST endpoint
	[ SUCCESS ]	Status code is 201
[ 412 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT endpoint
	[ SUCCESS ]	Status code is 201
[ 413 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Setup Bulk PUT /error endpoint
	[ SUCCESS ]	Status code is 201
[ 414 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxpayer callback - PARTICIPANT PUT Batch
	[ SUCCESS ]	Status code is 201
[ 415 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add ttkfxpayer callback - PARTICIPANT PUT Batch Error
	[ SUCCESS ]	Status code is 201
[ 416 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - PARTIES PUT ERROR
	[ SUCCESS ]	Status code is 201
[ 417 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS POST
	[ SUCCESS ]	Status code is 201
[ 418 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS PUT
	[ SUCCESS ]	Status code is 201
[ 419 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FX TRANSFERS ERROR
	[ SUCCESS ]	Status code is 201
[ 420 passed, 0 skipped, 0 failed of 460 ]
  Endpoints -> Add callback - FXQUOTES PUT
	[ SUCCESS ]	Status code is 201
[ 421 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Get Status Request before deposit
	[ SUCCESS ]	status to be 200
[ 422 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Deposit Funds in Settlement Account
	[ SUCCESS ]	status to be 202
[ 424 passed, 0 skipped, 0 failed of 460 ]
  Fundsin -> Get Status Request after deposit
	[ SUCCESS ]	status to be 200
	[ SUCCESS ]	Settlement Account Balance After FundsIn should be equal to the balance before plus the transfer amount
[ 425 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET PAYEEFSP/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 426 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET PAYERFSP/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 427 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET TESTFSP1/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 428 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET TESTFSP2/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 429 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET TESTFSP3/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 430 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> GET TESTFSP4/repository/parties    parties
	[ SUCCESS ]	resposne code 200
[ 431 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payee, Wallet] POST payeefsp/repository/parties  {{payeefspMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 432 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payee, Wallet] POST /ALS_host/participants {{payeefspMSISDN}}
	[ SUCCESS ]	resposne code is 202
[ 433 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payer, Wallet] POST /parties  {{payerfspMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 434 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payer, Wallet] POST /ALS_host/participants {{payerfspMSISDN}}
	[ SUCCESS ]	response code is 202
[ 435 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp1, Wallet] POST /parties  {{testfsp1MSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 436 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp1, Wallet] POST /ALS_host/participants {{testfsp1MSISDN}}
	[ SUCCESS ]	response code is 202
[ 437 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp1, Bank] POST parties  {{settlementtestfsp1bankMSISDN}} BANK
	[ SUCCESS ]	resposne code is 200
[ 438 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp1, Bank] POST /ALS_host/participants {{settlementtestfsp1bankMSISDN}}
	[ SUCCESS ]	response code is 202
[ 439 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp2, Wallet] POST /parties  {{testfsp2MSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 440 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp2, Wallet] POST /ALS_host/participants {{testfsp2MSISDN}}
	[ SUCCESS ]	response code is 202
[ 441 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp2, Bank] POST /parties  {{settlementtestfsp2bankMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 442 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp2, Bank] POST /ALS_host/participants {{settlementtestfsp2bankMSISDN}}
	[ SUCCESS ]	response code is 202
[ 443 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp3, Wallet] POST /parties  {{SIM3_MSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 444 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp3, Wallet] POST /ALS_host/participants {{SIM3_MSISDN}}
	[ SUCCESS ]	response code is 202
[ 445 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp3, Bank] POST /parties  {{settlementtestfsp3bankMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 446 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp3, Bank] POST /ALS_host/participants {{settlementtestfsp3bankMSISDN}}
	[ SUCCESS ]	response code is 202
[ 447 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp4, Wallet] POST /parties  {{SIM4_MSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 448 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp4, Wallet] POST /ALS_host/participants {{SIM4_MSISDN}}
	[ SUCCESS ]	response code is 202
[ 449 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp4, Bank] POST /parties  {{settlementtestfsp4bankMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 450 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [testfsp4, Bank] POST /ALS_host/participants {{settlementtestfsp4bankMSISDN}}
	[ SUCCESS ]	response code is 202
[ 451 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payee, no ext] POST /parties  {{settlementpayeefspNoExtensionMSISDN}}
	[ SUCCESS ]	resposne code is 200
[ 452 passed, 0 skipped, 0 failed of 460 ]
  Add Users to new Sims ; ALS registration -> [payee, no ext] POST /ALS_host/participants {{settlementpayeefspNoExtensionMSISDN}}
	[ SUCCESS ]	response code is 202
[ 453 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> GET /participants/limits before updating NDC
	[ SUCCESS ]	response code 200
[ 454 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/payerfsp/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 455 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/payeefsp/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 456 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/testfsp1/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 457 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/testfsp2/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 458 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/testfsp3/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 459 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> PUT /participants/testfsp4/limits adjust participants limits  NDC
	[ SUCCESS ]	response code 200
[ 460 passed, 0 skipped, 0 failed of 460 ]
  adjust ALL participants limits  NET DEBIT CAP (parametrized) -> GET /participants/limits after updating NDC
	[ SUCCESS ]	response code 200
[ 460 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding(MojaloopHub_Setup) -> Register MSISDN Oracle
[ 460 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding(MojaloopHub_Setup) -> Register BUSINESS Oracle
[ 460 passed, 0 skipped, 0 failed of 460 ]
  Oracle Onboarding(MojaloopHub_Setup) -> Register ALIAS Oracle

--------------------FINAL REPORT--------------------
Hub Account
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION - POST - /participants/{name}/accounts - [1/1]
	Hub Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Hub Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Hub Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Second Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION Second Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT CGS - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION CGS - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Third Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION Third Currency - POST - /participants/{name}/accounts - [1/1]
Settlement Models
	Create settlement model DEFERRED NET XXX - POST - /settlementModels - [1/1]
	Create settlement model DEFAULT DEFERRED NET - POST - /settlementModels - [1/1]
	Create settlement model Continous Gross - POST - /settlementModels - [1/1]
	Create settlement model Interchange Fee - POST - /settlementModels - [1/1]
Oracle Onboarding
	Register MSISDN Oracle - POST - /oracles - [1/1]
	Register BUSINESS Oracle - POST - /oracles - [1/1]
	Register ALIAS Oracle - POST - /oracles - [1/1]
Hub Account
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Second Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION Second Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_MULTILATERAL_SETTLEMENT Third Currency - POST - /participants/{name}/accounts - [1/1]
	Add Hub Account-HUB_RECONCILIATION Third Currency - POST - /participants/{name}/accounts - [1/1]
payerfsp account
	Add payerfsp - POST - /participants - [1/1]
	Add initial position and limits - payerfsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	payerfsp Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - payerfsp - POST - /participants/{name}/accounts/{id} - [1/1]
	payerfsp Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Add payeefsp with different currency for default settlement case - POST - /participants - [1/1]
	Add initial position and limits with different currency - payerfsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add payeefsp with different currency for default settlement case CGS - POST - /participants - [1/1]
	Add initial position and limits with different currency - payerfsp CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
payerfsp callbacks
	Add payerfsp callback - AUTHORIZATIONS - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - TXN REQUEST - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - BULK-TRANSFER POST - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - BULK-TRANSFER PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - BULK-TRANSFER ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
payerfsp notification_emails
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
payerfsp add_users
	Add SubId User - payerIdType - payerIdentifier - 100 - POST - /participants/{Type}/{ID}/{SubId} - [1/1]
payeefsp account
	Add payeefsp - POST - /participants - [1/1]
	Add initial position and limits - payeefsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	payeefsp Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - payeefsp - POST - /participants/{name}/accounts/{id} - [1/1]
	payeefsp Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Add payeefsp with different currency for default settlement case - POST - /participants - [1/1]
	Add initial position and limits with different currency - payeefsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add payeefsp with different currency for default settlement case CGS - POST - /participants - [1/1]
	Add initial position and limits with different currency - payeefsp CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
payeefsp callbacks
	Add payeefsp callback - AUTHORIZATIONS - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - TXN REQUEST - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - BULK-TRANSFER POST - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - BULK-TRANSFER PUT - POST - /participants/{name}/endpoints - [1/1]
	Add payeefsp callback - BULK-TRANSFER ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
payeefsp notification_emails
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
payeefsp oracle_registration
	Add payeeIdentifier in payeeIdType Oracle - POST - /participants/{Type}/{ID} - [1/1]
payeefsp add_users
	Add SubId User - payeeIdType - payeeIdentifier - 100 - POST - /participants/{Type}/{ID}/{SubId} - [1/1]
noresponsepayeefsp account
	Add noresponsepayeefsp - POST - /participants - [1/1]
	Add initial position and limits - noresponsepayeefsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	noresponsepayeefsp Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - noresponsepayeefsp - POST - /participants/{name}/accounts/{id} - [1/1]
	noresponsepayeefsp Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Add noresponsepayeefsp with different currency for default settlement case - POST - /participants - [1/1]
	Add initial position and limits with different currency - noresponsepayeefsp - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add noresponsepayeefsp with different currency for default settlement case CGS - POST - /participants - [1/1]
	Add initial position and limits with different currency - noresponsepayeefsp CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
noresponsepayeefsp callbacks
	Add noresponsepayeefsp callback - AUTHORIZATIONS - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - TXN REQUEST - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - BULK-TRANSFER POST - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - BULK-TRANSFER PUT - POST - /participants/{name}/endpoints - [1/1]
	Add noresponsepayeefsp callback - BULK-TRANSFER ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
noresponsepayeefsp notification_emails
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
testfsp1 account
	Add testfsp1 - POST - /participants - [1/1]
	Add initial position and limits - testfsp1 - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	testfsp1 Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - testfsp1 - POST - /participants/{name}/accounts/{id} - [1/1]
	testfsp1 Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Add testfsp1 with second currency - POST - /participants - [1/1]
	Add initial position and limits - testfsp1 with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add testfsp1 with second currency CGS - POST - /participants - [1/1]
	Add initial position and limits - testfsp1 with second currency CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
testfsp1 callbacks
	Add testfsp1 callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp1 callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
testfsp1 notification_emails
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
testfsp1 oracle_registration
	Add payeeIdentifier in payeeIdType Oracle - POST - /participants/{Type}/{ID} - [1/1]
testfsp1 add_users
	Add SubId User - testfsp1IdType - testfsp1Identifier - 100 - POST - /participants/{Type}/{ID}/{SubId} - [1/1]
testfsp2 account
	Add testfsp2 - POST - /participants - [1/1]
	Add initial position and limits - testfsp2 - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	testfsp2 Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - testfsp2 - POST - /participants/{name}/accounts/{id} - [1/1]
	testfsp2 Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Add testfsp2 with second currency - POST - /participants - [1/1]
	Add initial position and limits - testfsp2 with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add testfsp2 with second currency CGS - POST - /participants - [1/1]
	Add initial position and limits - testfsp2 with second currency CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
testfsp2 callbacks
	Add testfsp2 callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp2 callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
testfsp2 notification_emails
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
testfsp2 oracle_registration
	Add payeeIdentifier in payeeIdType Oracle - POST - /participants/{Type}/{ID} - [1/1]
testfsp2 add_users
	Add SubId User - testfsp2IdType - testfsp2Identifier - 100 - POST - /participants/{Type}/{ID}/{SubId} - [1/1]
Primary Account
	Add participant - POST - /participants - [1/1]
	Add initial position and limits - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Additional Currencies
	Add participant with second currency - POST - /participants - [1/1]
	Add initial position and limits with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add participant with second currency CGS - POST - /participants - [1/1]
	Add initial position and limits with second currency CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Endpoints
	Add callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - QUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk POST endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT /error endpoint - POST - /participants/{name}/endpoints - [1/1]
	Add testingtoolkitdfsp callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add testingtoolkitdfsp callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FXQUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
testingtoolkitdfsp fundsin
	testingtoolkitdfsp Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - testingtoolkitdfsp - POST - /participants/{name}/accounts/{id} - [1/1]
	testingtoolkitdfsp Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
ttkpayeefsp provisioning
	Add participant - POST - /participants - [1/1]
	Add initial position and limits - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - QUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk POST endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT /error endpoint - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add payerfsp callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add participant with second currency - POST - /participants - [1/1]
	Add initial position and limits with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add participant with second currency CGS - POST - /participants - [1/1]
	Add initial position and limits with second currency CGS - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	Add payeeIdentifier in payeeIdType Oracle - POST - /participants/{Type}/{ID} - [1/1]
	Add payeeIdentifier in payeeIdType Oracle - POST - /participants/{Type}/{ID} - [1/1]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp3
	Add testfsp3 - POST - /participants - [1/1]
	[NBC=0]Add initial position and limits - testfsp3 - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	GET /participants testfsp3 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - testfsp3 - POST - /participants/{name}/accounts/{id} - [1/1]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-callbacks
	Add testfsp3 callback - AUTHORIZATIONS - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - TXN REQUEST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - TRANSFER POST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - TRANSFER PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp3 callback - TRANSFER ERROR - POST - /participants/{name}/endpoints - [1/1]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp3-notifications
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [0/0]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [0/0]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [0/0]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp4
	Add testfsp4 - POST - /participants - [1/1]
	[NBC=0]Add initial position and limits - testfsp4 - POST - /participants/{name}/initialPositionAndLimits - [1/1]
	GET /participants testfsp4 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - testfsp4 - POST - /participants/{name}/accounts/{id} - [1/1]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-callbacks
	Add testfsp4 callback - AUTHORIZATIONS - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - PARTIES PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - QUOTES - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - TXN REQUEST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - TRANSFER POST - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - TRANSFER PUT - POST - /participants/{name}/endpoints - [1/1]
	Add testfsp4 callback - TRANSFER ERROR - POST - /participants/{name}/endpoints - [1/1]
FSP Onboarding(MojaloopSims_Onboarding)-testfsp4-notifications
	Set Email-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [0/0]
	Set Email-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [0/0]
	Set Email-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [0/0]
Primary Account
	Add participant - POST - /participants - [1/1]
	Add initial position and limits - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Additional Currencies
	Add participant with second currency - POST - /participants - [1/1]
	Add initial position and limits with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Endpoints
	Add callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - QUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk POST endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT /error endpoint - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxp1 callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxp1 callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FXQUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
Fundsin
	Get Status Request before deposit for currency2 - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account currency2 - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit for currency2 - GET - /participants/{name}/accounts - [2/2]
Primary Account
	Add participant with second currency - POST - /participants - [1/1]
	Add initial position and limits with second currency - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Endpoints
	Add callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - QUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk POST endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT /error endpoint - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxpayee callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxpayee callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FXQUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
Primary Account
	Add participant - POST - /participants - [1/1]
	Add initial position and limits - POST - /participants/{name}/initialPositionAndLimits - [1/1]
Endpoints
	Add callback - PARTICIPANT PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - QUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID PUT Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTICIPANT SUB-ID DELETE - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID GET - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES SUB-ID ERROR PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-NET_DEBIT_CAP_ADJUSTMENT_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Set Endpoint-SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	DFSP Endpoint-NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk POST endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT endpoint - POST - /participants/{name}/endpoints - [1/1]
	Setup Bulk PUT /error endpoint - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxpayer callback - PARTICIPANT PUT Batch - POST - /participants/{name}/endpoints - [1/1]
	Add ttkfxpayer callback - PARTICIPANT PUT Batch Error - POST - /participants/{name}/endpoints - [1/1]
	Add callback - PARTIES PUT ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS POST - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS PUT - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FX TRANSFERS ERROR - POST - /participants/{name}/endpoints - [1/1]
	Add callback - FXQUOTES PUT - POST - /participants/{name}/endpoints - [1/1]
Fundsin
	Get Status Request before deposit - GET - /participants/{name}/accounts - [1/1]
	Deposit Funds in Settlement Account - POST - /participants/{name}/accounts/{id} - [1/1]
	Get Status Request after deposit - GET - /participants/{name}/accounts - [2/2]
Add Users to new Sims ; ALS registration
	GET PAYEEFSP/repository/parties    parties - GET - /repository/parties - [1/1]
	GET PAYERFSP/repository/parties    parties - GET - /repository/parties - [1/1]
	GET TESTFSP1/repository/parties    parties - GET - /repository/parties - [1/1]
	GET TESTFSP2/repository/parties    parties - GET - /repository/parties - [1/1]
	GET TESTFSP3/repository/parties    parties - GET - /repository/parties - [1/1]
	GET TESTFSP4/repository/parties    parties - GET - /repository/parties - [1/1]
	[payee, Wallet] POST payeefsp/repository/parties  {{payeefspMSISDN}} - POST - /repository/parties - [1/1]
	[payee, Wallet] POST /ALS_host/participants {{payeefspMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[payer, Wallet] POST /parties  {{payerfspMSISDN}} - POST - /repository/parties - [1/1]
	[payer, Wallet] POST /ALS_host/participants {{payerfspMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp1, Wallet] POST /parties  {{testfsp1MSISDN}} - POST - /repository/parties - [1/1]
	[testfsp1, Wallet] POST /ALS_host/participants {{testfsp1MSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp1, Bank] POST parties  {{settlementtestfsp1bankMSISDN}} BANK - POST - /repository/parties - [1/1]
	[testfsp1, Bank] POST /ALS_host/participants {{settlementtestfsp1bankMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp2, Wallet] POST /parties  {{testfsp2MSISDN}} - POST - /repository/parties - [1/1]
	[testfsp2, Wallet] POST /ALS_host/participants {{testfsp2MSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp2, Bank] POST /parties  {{settlementtestfsp2bankMSISDN}} - POST - /repository/parties - [1/1]
	[testfsp2, Bank] POST /ALS_host/participants {{settlementtestfsp2bankMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp3, Wallet] POST /parties  {{SIM3_MSISDN}} - POST - /repository/parties - [1/1]
	[testfsp3, Wallet] POST /ALS_host/participants {{SIM3_MSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp3, Bank] POST /parties  {{settlementtestfsp3bankMSISDN}} - POST - /repository/parties - [1/1]
	[testfsp3, Bank] POST /ALS_host/participants {{settlementtestfsp3bankMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp4, Wallet] POST /parties  {{SIM4_MSISDN}} - POST - /repository/parties - [1/1]
	[testfsp4, Wallet] POST /ALS_host/participants {{SIM4_MSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[testfsp4, Bank] POST /parties  {{settlementtestfsp4bankMSISDN}} - POST - /repository/parties - [1/1]
	[testfsp4, Bank] POST /ALS_host/participants {{settlementtestfsp4bankMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
	[payee, no ext] POST /parties  {{settlementpayeefspNoExtensionMSISDN}} - POST - /repository/parties - [1/1]
	[payee, no ext] POST /ALS_host/participants {{settlementpayeefspNoExtensionMSISDN}} - POST - /participants/{Type}/{ID} - [1/1]
adjust ALL participants limits  NET DEBIT CAP (parametrized)
	GET /participants/limits before updating NDC - GET - /participants/limits - [1/1]
	PUT /participants/payerfsp/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	PUT /participants/payeefsp/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	PUT /participants/testfsp1/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	PUT /participants/testfsp2/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	PUT /participants/testfsp3/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	PUT /participants/testfsp4/limits adjust participants limits  NDC - PUT - /participants/{name}/limits - [1/1]
	GET /participants/limits after updating NDC - GET - /participants/limits - [1/1]
Oracle Onboarding(MojaloopHub_Setup)
	Register MSISDN Oracle - POST - /oracles - [0/0]
	Register BUSINESS Oracle - POST - /oracles - [0/0]
	Register ALIAS Oracle - POST - /oracles - [0/0]
--------------------FINAL REPORT--------------------

Test Suite:Standard Provisioning Collection
Environment:Development
┌───────────────────────────────────────────────────┐
│                      SUMMARY                      │
├───────────────────┬───────────────────────────────┤
│ Total assertions  │ 460                           │
├───────────────────┼───────────────────────────────┤
│ Passed assertions │ 460                           │
├───────────────────┼───────────────────────────────┤
│ Failed assertions │ 0                             │
├───────────────────┼───────────────────────────────┤
│ Total requests    │ 455                           │
├───────────────────┼───────────────────────────────┤
│ Total test cases  │ 49                            │
├───────────────────┼───────────────────────────────┤
│ Passed percentage │ 100.00%                       │
├───────────────────┼───────────────────────────────┤
│ Started time      │ Wed, 10 Sep 2025 11:34:42 GMT │
├───────────────────┼───────────────────────────────┤
│ Completed time    │ 2025-09-10T11:38:58.848Z      │
├───────────────────┼───────────────────────────────┤
│ Runtime duration  │ 255947 ms                     │
└───────────────────┴───────────────────────────────┘
/tmp/TTK-Assertion-Report-standard_provisioning_collection-2025-09-10T11:38:58.848Z.html was generated
Report saved on TTK backend server successfully and is available at testing-toolkit.local/api/history/test-reports/standard_provisioning_collection_2025-09-10T11:38:58.848Z?format=html
No Slack webhook URLs configured.
Terminate with exit code 0
Test Runner finished with exit code: 0

```
