# Kubernetes Cluster Deployment on Ubuntu 22.04 LTS Server

This guide will walk you through the process of deploying a Kubernetes cluster on Ubuntu 22.04 LTS servers. Before you begin, make sure you have the following prerequisites in place:

- Three Ubuntu 22.04 LTS servers, each with a non-root user with sudo privileges.
- A fully-qualified domain name (FQDN) for each server.
- The servers should be able to communicate with each other over a private network.

---

## Update and Upgrade Servers

To ensure your servers are up to date, run the following commands:

```bash
sudo apt update && sudo apt full-upgrade -y
```

---

## Set Up Fully-Qualified Hostnames on All Nodes

Set the fully-qualified hostname on each node as follows:

### Run on `master1`:

```bash
sudo hostnamectl set-hostname master1.example.com
```

### Run on `worker1`:

```bash
sudo hostnamectl set-hostname worker1.example.com
```

### Run on `worker2`:

```bash
sudo hostnamectl set-hostname worker2.example.com
```

---

## Configure Host Entries

Edit the `/etc/hosts` file on each node to include the following entries:

```plaintext
10.0.100.138    master1.example.com    master1
10.0.100.216    worker1.example.com    worker1
10.0.100.218    worker2.example.com    worker2
```

---

## Disable Swap and Configure Kernel Parameters

Disable swap and add kernel parameters on each node with the following commands:

```bash
sudo swapoff -a
```

Comment out the swap in the fstab file:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Load kernel modules:

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure sysctl settings:

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply sysctl settings:

```bash
sudo sysctl --system
```

---

## Install Containerd Runtime

Install the Containerd runtime on each node with the following commands:

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

```bash
sudo apt update && sudo apt install containerd.io -y
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

---

## Add Repository for Kubernetes

Add the Kubernetes repository with the following command:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/kubernetes-xenial.gpg
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

---

## Install kubelet, kubeadm, and kubectl

Install the Kubernetes components with the following commands:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Initialize Kubernetes Cluster (Master Only)

Initialize the Kubernetes cluster on the master node with the following command:

```bash
sudo kubeadm init --control-plane-endpoint=master1.example.com --pod-network-cidr=192.168.0.0/16
```

---

## Add Kube Config File to User's Environment

To access the Kubernetes cluster using `kubectl`, perform the following steps:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Join Other Nodes to the Cluster (Workers)

Join the worker nodes to the cluster with the following command:

```bash
sudo kubeadm join master1.coe.com:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx
```

Note: You can find the token and keys from the `kubeadm init` output.

---

## Install Calico CNI

Install the Calico CNI with the following commands:

```bash
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

---

This guide provides the necessary steps to deploy a Kubernetes cluster on Ubuntu 22.04 LTS servers. Follow these steps carefully to set up your Kubernetes environment successfully.
