### Pre-Requisistes

- Access to cloud platform
- Permissions to create VM instances using RedHat or CentOS based distributions
- Permissions to SSH
- VPC with at least 1 subnet with outbound access to internet
- Number of available IPs in subnet/s depending on the size of target cluster
- Security Groups with necessary open ports mentioned here 
  https://kubernetes.io/docs/reference/ports-and-protocols/. You may create separate security groups for
  master node and worker node.
  Also make sure TCP port 179 is opened for both worker 
  and master nodes for calico BGP connectivity to work.
  calico-nodes will fail the readiness probe without it.
  

### Step 1 (Boostrapping of the nodes)

Spin up at least 2 VM instances using following script as `user data`  

```shell script
#!/bin/bash

K8S_VERSION_TO_INSTALL=1.23.3

#Disable SWAP
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum update

#Install containerd
sudo yum install -y containerd
sudo systemctl enable containerd
sudo systemctl start containerd

#iptables to be able to see bridged traffic
sudo modprobe br_netfilter

sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

#Configure yum repo for k8s packages
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

#Install kubeadm,kubelet and kubectl
sudo yum install -y kubeadm-$K8S_VERSION_TO_INSTALL kubectl-$K8S_VERSION_TO_INSTALL kubelet-$K8S_VERSION_TO_INSTALL --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

```

### Step 2 (Setup of master node)
- SSH into target master node
- Execute `kubeadm init --pod-network-cidr=192.168.0.0/16`
- Copy and save output of `kubeadm init`. You will need to execute it from worker node.
- Execute below script
```shell script
sudo su
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
```

### Step 3 (Setup of worker node)

- SSH into targer worker node
- Execute `kubeadm join ..` command appeared in `kubeadm init` operation on master node
- Execute below script
```shell script
sudo su
mkdir ~/.kube
cp /etc/kubernetes/kubelet.conf ~/.kube/config
```

### Step 4 (Install CNI plugin)

- From the master node execute 
`kubectl create -f https://docs.projectcalico.org/v3.19/manifests/calico.yaml`