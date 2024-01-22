![Static Badge](https://img.shields.io/badge/Ubuntu-22.04.3-%23E95420?style=flat&logo=ubuntu)
![Static Badge](https://img.shields.io/badge/Docker-25.0.0-%232496ED?style=flat&logo=docker)
![Static Badge](https://img.shields.io/badge/Kubernetes-1.29.1-%23326CE5?style=flat&logo=Kubernetes)

# k8s cluster setting

This repository is a guide on how to build a Kubernetes cluster using Oracle virtual machines. Detailed virtual machine specifications and setup methods will be explained below.

## Contents

1. [VM Spec](#vm_spec)
2. [VM Setting](#vm_setting)
3. [Docker install](#docker_install)
4. [Kubernetes install](#k8s_install)
5. [Control-plane configuration](#control_plane)

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
    * IP : 192.168.200.100 
    * Port : 1000
* worker node 1 
    * IP : 192.168.200.101
    * Port : 1001
* worker node 2 
    * IP : 192.168.200.102
    * Port : 1002

<a name='vm_setting'></a>

## 2. VM Setting
First, we will create a master virtual machine and name it ‘k8s-master’. Then install Ubuntu Live Server on this virtual machine.

Second, modify the `/etc/netplan/00-installer-config.yaml` file as follows.

```bash
sudo vi /etc/netplan/00installer-config.yaml
```

```yaml
# This is the network config written by 'subiquity'
network:
    ethernets:
        enp0s3:
            addresses:
                - 192.168.200.100/24
            routes:
                - to: default
                  via: 192.168.200.1
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
192.168.200.100 k8s-master
192.168.200.101 k8s-node1
192.168.200.102 k8s-node2
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

Applies to node2 as well.

<a name='docker_install'></a>

## 3. Docker Install

To use a Kubernetes cluster, Docker must be installed on all virtual machines.

Simply copy and paste the Docker installation command to any virtual machine. The installation commands are well explained on the [Docker site](https://docs.docker.com/engine/install/ubuntu/).

Once docker installation is complete, check the version by entering the command.
If the version appears correctly, installation is complete.

```bash
$ docker --version
```

<a name='k8s_install'></a>

## 4. Kubernetes Install

You must terminate swap before installing Kubernetes.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*/)$/#1/g' /etc/fstab
```

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

Now let's install Kubernetes.

Detailed information regarding installation is on the [kubernetes site](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

You've completed the installation, start the kubelet

```bash
[All VM]
sudo systemctl start kubelet
sudo systemctl enable kubelet
```

<a name='control_plane'></a>

## 5. Control-plane configuration

You must typing `k8s-master`VM.
```bash
master@k8s-master: ~$ kubeadm init
```

When the kubeadm command is finished, a token appears at the end. Copy this and save it.

I will save this token in token.txt on the master vm.

```bash
master@k8s-master: ~$ cat > token.txt
> kubeadm join 192.168.200.100 ...
```
Next, set user permissions to use the cluster.
```bash
master@k8s-master: ~$ sudo mkdir -p $HOME/.kube
master@k8s-master: ~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
master@k8s-master: ~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify that it is operating correctly.

```bash
master@k8s-master: ~$ kubectl get nodes
```

Next, Installing a Pod network add-on
```bash
master@k8s-master: ~$ kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s-1.11.yaml
```
