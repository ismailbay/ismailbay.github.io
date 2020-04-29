---
layout: post
title: "Creating a multi-node Kubernetes Cluster on Fedora"
tags:
- Kubernetes
- Linux
- DevOps
thumbnail_path: blog/kubernetes/kubernetes-stacked-color.png
---

## 1. Prerequisites
 * Follow the [Getting Started with Virtualization](https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-virtualization/) until the networking part
 * configure [Bridge Networking on Fedora](https://lukas.zapletalovi.com/2015/09/fedora-22-libvirt-with-bridge.html)
 * if you are planning to create more than one node, it is easier to fully configure the first node. Then you can clone it before joining it to the cluster: [how to clone virtual machines](https://www.cyberciti.biz/faq/how-to-clone-existing-kvm-virtual-machine-images-on-linux/). 
  * after cloning you must ssh into the clone and [adapt hostname](https://linuxize.com/post/how-to-change-hostname-on-ubuntu-18-04/) and network settings under `/etc/netplan/`

## 2. Virtual Machines
 * use the gist to create the virtual machines
 * during installation use the name **k8s-master** on the first machine and **k8s-node0x** on the nodes
 * install `OpenSSH Server` package
 * reboot
 * call `sudo nmap -sP 192.168.1.0/24` on your host to get the ip address of the newly created VM:

```
Nmap scan report for 192.168.1.208
Host is up (0.00024s latency).
MAC Address: 52:54:00:A4:6B:0B (QEMU virtual NIC)
```

## 3. Post-Install
ssh into the VM

### packages
 * `sudo apt update && sudo apt upgrade`
 * `sudo apt install curl git gnupg vim # and other packages you want`

### swap
 * `sudo swapoff -a`
 * `sudo sed -i '/ swap / s/^/#/' /etc/fstab`

### static ip configuration
edit `/etc/netplan/01-netcfg.yaml`:

```
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses: [192.168.1.201/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1,8.8.8.8]
```

`sudo netplan apply`


## 4. Hostnames and IPs
copy to your `/etc/hosts` on the host machine

```
192.168.1.201   k8s-master
192.168.1.211   k8s-node01
192.168.1.212   k8s-node02
192.168.1.213   k8s-node03
```

## 5. SSH

### Keys
`ssh-keygen -t rsa`
`ssh-copy-id <user>@k8s-master` and on other nodes

### Config
```
Host k8s-master
        Hostname k8s-master
        IdentityFile /home/ibay/.ssh/id_rsa_k8s
Host k8s-node01
        Hostname k8s-node01
        IdentityFile /home/ibay/.ssh/id_rsa_k8s
Host k8s-node02
        Hostname k8s-node02
        IdentityFile /home/ibay/.ssh/id_rsa_k8s
Host k8s-node03
        Hostname k8s-node03
        IdentityFile /home/ibay/.ssh/id_rsa_k8s
```

`chmod 600 ~/.ssh/config`

## 6. Install the components
on each VM

 * `sudo apt update && sudo apt upgrade`
 * `sudo apt install -qy apt-transport-https software-properties-common`
 * `curl -s https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
 * `sudo add-apt-repository  "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`
 * `sudo apt update && sudo apt install -qy docker-ce`
 * `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`
 * `echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list`
 * `sudo apt update && sudo apt install -y kubeadm kubelet kubectl`
 * `sudo systemctl enable docker`
 * `sudo systemctl start docker`

## 7. Init k8s master
`sudo kubeadm init`

copy the output with the token. It will be needed to join the workers:

```
kubeadm join 192.168.1.201:6443 --token v15nn2.efdd86ctvv2py4sj -- \
discovery-token-ca-cert-hash sha256:1501b51b6052216b206a5f065b50617a3f5bbca57f3b40d60314c04a27e4f7e7
```

`mkdir -p $HOME/.kube`

`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

### install networking
on master install one of the kubernetes network solutions, e.g. Weave Net 

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

## 8. Join the workers to the cluster
```
kubeadm join 192.168.1.201:6443 --token v15nn2.efdd86ctvv2py4sj \
--discovery-token-ca-cert-hash sha256:1501b51b6052216b206a5f065b50617a3f5bbca57f3b40d60314c04a27e4f7e7
```
 
## 9. Test your cluster
 * copy the kube config to your host: `scp k8s-master:/home/ibay/.kube/config ~/.kube`
 * check your cluster: `kubectl get nodes`

deploy hello-kubernetes with replication 3 or the amount of your nodes

```
curl -L https://raw.githubusercontent.com/paulbouwer/hello-kubernetes/master/yaml/hello-kubernetes.yaml \
  | sed 's/replicas: .*/replicas: 3/' \
  | kubectl apply -f -
```
