![Static Badge](https://img.shields.io/badge/Ubuntu-22.04.3-%23E95420?style=flat&logo=ubuntu)
![Static Badge](https://img.shields.io/badge/Kubernetes-1.30.2-%23326CE5?style=flat&logo=Kubernetes)
![Static Badge](https://img.shields.io/badge/CRIO-1.24.6-%23326CE5?style=flat)

# k8s cluster setting

This repository is a guide on how to build a Kubernetes cluster using Oracle virtual machines. Detailed virtual machine specifications and setup methods will be explained below.

## Contents

1. [VM Spec](#vm_spec)
2. [VM Setting](#vm_setting)
3. [Container runtime interface install ( CRI-O )](#crio)
4. [Kubernetes install](#k8s_install)
5. [Kubernetes Cluster](#k8s-cluster)
6. [ERROR FIX](#error_fix)

<a name='vm_spec'></a>

## 1. VM Spec

We will create three virtual machines.
```
master node, Worker node Spec
    - Processor : 2
    - System memory : 4096 MB
    - Vedio memory : 16 MB
```
The virtual IPs and ports we will use when building a cluster are as follows.

|Node name|Node IP|Node Port|
|:---:|:---:|:---:|
|***k8s-master***|192.168.1.10|1000|
|***k8s-node1***|192.168.1.10|1000|
|***k8s-node2***|192.168.1.10|1000|
|***k8s-node3***|192.168.1.10|1000|

<a name='vm_setting'></a>

## 2. VM Setting
First, we will create a master virtual machine and name it ‘k8s-master’. Then install Ubuntu Live Server on this virtual machine.

Second, modify the `/etc/netplan/00-installer-config.yaml` file as follows.

```bash
sudo vi /etc/netplan/00-installer-config.yaml
```

```yaml
# This is the network config written by 'subiquity'
network:
    ethernets:
        enp0s3:
            addresses:
                - 192.168.1.10/24
            routes:
                - to: default
                  via: 192.168.1.1
            nameservers:
                addresses:
                    - 8.8.8.8
                search:
                    - 8.8.4.4
    version: 2
```

And then apply netplan.

```bash
sudo netplan apply
```
If you want to check the changed results, try using the `ifconfig` command.

Third, set up the hosts.

```bash
sudo vi /etc/hosts
```

```vim
127.0.0.1 localhost
192.168.1.10 k8s-master
192.168.1.11 k8s-node1
192.168.1.12 k8s-node2
192.168.1.12 k8s-node3
```

Then, shut down the virtual machine and create two copies of the master virtual machine.

Finally, change the IP and hostname of the newly created virtual machine.
```bahs
master@k8s-node1: ~$ sudo vi /etc/netplan/00installer-config.yaml
```
```bash
master@k8s-node1: ~$ sudo vi /etc/hostname
```

`hostname`

```bash
k8s-node1
```

```bash
master@k8s-node1: ~$ sudo hostname -F /etc/hostname
master@k8s-node1: ~$ sudo reboot
```

Applies to node2, node3 as well.

<a name='crio'></a>

## 3. Container runtime interface install ( CRI-O )

If you are using k8s version 1.20.x or later, you will need to install Container runtime interface (CRI).

CRI is a standard interface called CRI that allows communication with multiple container runtimes.

We will install and use CRI-O among several CRIs.

Change to root privileges
```bash
master@k8s-master:~$ sudo -i
```

First, set up the network settings.

```bash
root@k8s-master:~$ cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
```

```bash
root@k8s-master:~$ sudo modprobe overlay
root@k8s-master:~$ sudo modprobe br_netfilter
```

```bash
root@k8s-master:~$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```bash
root@k8s-master:~$ sudo sysctl --system
```

CRI-O install.

```bash
root@k8s-master:~$ export OS=xUbuntu_22.04 # OS version
root@k8s-master:~$ export VERSION=1.24 # CRI-O version
```

```bash
root@k8s-master:~$ echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
root@k8s-master:~$ echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
```

```bash
root@k8s-master:~$ curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
root@k8s-master:~$ curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -
```

```bash
root@k8s-master:~$ sudo apt-get update

root@k8s-master:~$ sudo apt-get -y install cri-o cri-o-runc cri-tools

root@k8s-master:~$ sudo systemctl daemon-reload
root@k8s-master:~$ sudo systemctl enable crio --now
```

<a name='k8s_install'></a>

## 4. Kubernetes Install

For more information, please visit the Kubernetes homepage.

### [Kubernetes install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

```bash
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl gpg

$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl

$ sudo systemctl enable --now kubelet
```

<a name='k8s-cluster'></a>

## 5. Kubernetes Cluster 

```bash
root@k8s-master:~$ kubeadm init
```

Running the kubeadm command generates relevant setup commands and a cluster configuration token.

```bash
root@k8s-master:~$ mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
root@k8s-node{1~3}:~$ kubeadm join 192.168.1.10:6443 --token [token..]
```

Grant kubectl commands to regular users as well
```bash
master@k8s-master:~$ mkdir -p $HOME/.kube
master@k8s-master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
master@k8s-master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
<a name='vm_spec'></a>

## 6. ERROR FIX

### 6-1. You must perform this command if your Kubernetes API server is unstable.

```bash
$ containerd config default | tee /etc/containerd/config.toml
$ sed -i 's/SystemdCgroup = False/SystemdCgroup = true/g' /etc/containerd/config.toml
$ service containerd restart
$ service kubelet restart
```
I believe this issue is caused by Docker and Containerd crashing while running.

### 6-2. If node is not ready state.
```bash
$ systemctl restart kubelet
$ systemctl restart containerd
```