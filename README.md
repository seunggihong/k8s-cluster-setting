![Static Badge](https://img.shields.io/badge/Ubuntu-22.04.3-%23E95420?style=flat&logo=ubuntu)
![Static Badge](https://img.shields.io/badge/Docker-25.0.0-%232496ED?style=flat&logo=docker)
![Static Badge](https://img.shields.io/badge/Kubernetes-1.29.1-%23326CE5?style=flat&logo=Kubernetes)

# k8s cluster setting

This repository is a guide on how to build a Kubernetes cluster using Oracle virtual machines. Detailed virtual machine specifications and setup methods will be explained below.

## Contents

1. [VM Spec](#vm_spec)
2. [VM Setting](#vm_setting)
3. [Kubernetes install](#k8s_install)
4. [Control-plane configuration](#control_plane)
5. [ERROR FIX](#error_fix)
6. [++Docker install](#docker_install)

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

* master 
    * IP : 192.168.1.10 
    * Port : 1000
* worker node 1 
    * IP : 192.168.1.11
    * Port : 1001
* worker node 2 
    * IP : 192.168.1.12
    * Port : 1002
* worker node 3
    * IP : 192.168.1.13
    * Port : 1003

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
master@k8s-node1: ~$ sudo hostname apply
master@k8s-node1: ~$ sudo reboot
```

Applies to node2, node3 as well.

<a name='k8s_install'></a>

## 3. Kubernetes Install

CRI-O instll

cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

sudo -i

export OS=xUbuntu_20.04 # OS 버전
export VERSION=1.19     # cri-o 버전

echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

sudo apt-get update

sudo apt-get -y install cri-o cri-o-runc cri-tools

sudo systemctl daemon-reload
sudo systemctl enable crio --now


k8s install

sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

권한 부여
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

<a name='vm_spec'></a>

## 5. ERROR FIX

### 5-1. You must perform this command if your Kubernetes API server is unstable.

```bash
$ containerd config default | tee /etc/containerd/config.toml
$ sed -i 's/SystemdCgroup = False/SystemdCgroup = true/g' /etc/containerd/config.toml
$ service containerd restart
$ service kubelet restart
```
I believe this issue is caused by Docker and Containerd crashing while running.

### 5-2. If node is not ready state.
```bash
$ systemctl restart kubelet
$ systemctl restart containerd
```

<a name='docker_install'></a>

## 6. ++Docker Install

To use a Kubernetes cluster, Docker must be installed on all virtual machines.

Simply copy and paste the Docker installation command to any virtual machine. The installation commands are well explained on the [Docker site](https://docs.docker.com/engine/install/ubuntu/).

Once docker installation is complete, check the version by entering the command.
If the version appears correctly, installation is complete.

```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
$ sudo systemctl restart docker
$ sudo systemctl enable docker
```

```bash
$ docker --version
```

Because Kubernetes ended native support for the Docker container runtime after version 1.24, you will need to additionally install cri-docker to connect Docker and Kubernetes.

```bash
$ git clone https://github.com/Mirantis/cri-dockerd.git
$ wget https://storage.googleapis.com/golang/getgo/installer_linux
$ chmod +x ./installer_linux
./installer_linux
$ source ~/.bash_profile
$ cd cri-dockerd
$ mkdir bin
$ go build -o bin/cri-dockerd
$ mkdir -p /usr/local/bin
$ install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
$ cp -a packaging/systemd/* /etc/systemd/system
$ sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

$ sudo systemctl daemon-reload
$ sudo ystemctl enable cri-docker.service
$ sudo systemctl enable --now cri-docker.socket
$ sudo systemctl restart docker && sudo systemctl restart cri-docker
$ sudo systemctl status cri-docker.socket --no-pager
```

Docker Cgroup Change.
```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```