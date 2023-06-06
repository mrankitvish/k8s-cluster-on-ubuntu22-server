# Kubernetes Deployment on Ubuntu 22.04 LTS
## Prerequisites
#### Before you begin, you should have the following:

- Three Ubuntu 22.04 LTS servers, each with a non-root user with sudo privileges.
- A fully-qualified domain name (FQDN) for each server.
- The servers should be able to communicate with each other over a private network.
---
#### Update and upgrade servers

```
sudo apt update && sudo apt full-upgrade -y
```
#### Set up fully-qualified hostname on all nodes

##### Run at master1
```
sudo hostnamectl set-hostname master1.example.com
```

##### Run at worker1
```
sudo hostnamectl set-hostname worker1.example.com
```

##### Run at worker2
```
sudo hostnamectl set-hostname worker2.example.com
```

####  Do host entry in `/etc/hosts` file of each node
```
10.0.100.138    master1.example.com    master1   
10.0.100.216    worker1.example.com    worker1
10.0.100.218    worker2.example.com    worker2
```

#### Disable swap and add some kernel parameters on each node
```
sudo swapoff -a
```
Comment out the swap in the fstab file.
```
mount -a
```
```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

```
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
```
sudo sysctl --system
```

#### Install containerd runtime on each node

```
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
```

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
```
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
```
sudo apt update && sudo apt install containerd.io
```
```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```
```
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
```
sudo systemctl restart containerd && sudo systemctl enable containerd
```

#### Adding repo for Kubernetes:
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/kubernetes-xenial.gpg
```
```
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

#### Install kubelet, kubeadm, and kubectl:

```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
#### Initialize Kubernetes cluster (master only)
```
sudo kubeadm init --control-plane-endpoint=master1.example.com --pod-network-cidr=192.168.0.0/16
```

#### Add kube config file to user’s environment to get access the kubectl command
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Join other nodes to cluster (workers)
```
sudo kubeadm join master1.coe.com:6443 --token xxxxx \
--discovery-token-ca-cert-hash sha256:xxxxx
```
#### `Note`: You shall find token and keys from the `kubeadm init` output.

#### Install Calico CNI
```
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
```
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
```
```
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
```
```
kubectl apply -f custom-resources.yaml
```
