---
layout: post
title: Setting up Kubernetes in Ubuntu 
tags: linux kubernetes
---

# Setting up Kubernetes in Ubuntu

## Install Docker 
Docker needs to be installed on all the nodes. Master and worker nodes.

### Packages to be installed
```
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt-cache policy docker-ce
sudo apt install docker-ce
```

### Verify status
```
sudo systemctl status docker

docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2020-05-19 17:00:41 UTC; 17s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 24321 (dockerd)
      Tasks: 8
     Memory: 46.4M
     CGroup: /system.slice/docker.service
             └─24321 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
### permissions for user
```
sudo usermod -aG docker ${USER}
su - ${USER}
``` 
logout and log back in

if you need to explicitely add to docker group then perform the below 

```
sudo usermod -aG docker username
```
### Test
Test by running `docker ps` and ensure no error is returned.

### Modify cgroup driver

modify the file `/etc/docker/daemon.json` to the below contents. if doesn't exist, create it. This is needed because systemd is recommended.
```
{ 
    "exec-opts": ["native.cgroupdriver=systemd"],
    "storage-driver": "overlay2"
}
```

#### Restart docker

```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
Ensure docker service is running at this stage.

## Install Kubernetes 
### Apt package updates
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### Public key and update repository
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install kubernetes 
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Install the above on master and worker nodes. The worker node does not need kubectl, but needs kubeadm and kubelet.

### Disable swap

Swap needs to be turned off on all the master and worker nodes. If not disabled when creating the cluster, kubelet does not come up 
```
swapoff -a
```
To permanently turn it off, edit the swap line from /etc/fstab

### changing DNS
in ubuntu 20.02, netplan has to be used and DNS resolver uses a stub

add entries into /var/run/systemd/resolve/resolv.conf 

editing directly into /etc/resolv.conf will be of no use.

```
root@k8s-master:~# more /var/run/systemd/resolve/resolv.conf
# This file is managed by man:systemd-resolved(8). Do not edit.
#
# This is a dynamic resolv.conf file for connecting local clients directly to
# all known uplink DNS servers. This file lists all configured search domains.
#
# Third party programs must not access this file directly, but only through the
# symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
# replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 8.8.8.8
```

#### Example of netplan entries [optional]
if you need to add persistent IP address on ubuntu 20.04, netplan has to be used. below is an example

create netplan file on /etc/netplan/00-installer-config.yaml

```
root@k8s-master:~# more /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      addresses:
      - 10.85.47.160/24
      gateway4: 10.85.47.129
      dhcp4: no
      nameservers:
        addresses:
        - 8.8.8.8
      optional: true
    ens6f0:
      addresses:
      - 192.170.1.1/24
      dhcp4: no
      nameservers:
        addresses:
        - 8.8.8.8
      optional: true
    lo:
      addresses:
      - 127.0.0.1/32
      - 100.1.1.1/32
```

#### Apply netplan
```
sudo netplan apply 
```

### Setup hostnames
setup hostnames for master and worker nodes and ensure they are reffered in the `/etc/hosts` file.

Here is an example on the worker node. Similarly things should be present on the other nodes.
```
root@k8s-worker1:~# more /etc/hosts
127.0.0.1	localhost
10.85.47.165	k8s-worker1
10.85.47.160	k8s-master
10.85.47.166	k8s-worker2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
10.85.47.165 ubuntu
10.85.47.165 ubuntu
```

### Intialize the cluster
```
root@k8s-master:~# kubeadm init --pod-network-cidr=10.244.0.0/16
```
once completed run the below so that kubectl commands could be run. This is basically exporting the env/profile variables

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install CNI 
Run the below to setup the 

```
root@k8s-master:~# kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

older versions of flannel needed separate RBAC also

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

### Verify pods
```
root@k8s-master:~# kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-64897985d-lhnhn              1/1     Running   0          23h
coredns-64897985d-x929d              1/1     Running   0          23h
etcd-k8s-master                      1/1     Running   7          23h
kube-apiserver-k8s-master            1/1     Running   1          23h
kube-controller-manager-k8s-master   1/1     Running   0          23h
kube-flannel-ds-2db92                1/1     Running   0          23h
kube-flannel-ds-9vhz2                1/1     Running   0          23h
kube-flannel-ds-r4lfv                1/1     Running   0          23h
kube-proxy-ktvcx                     1/1     Running   0          23h
kube-proxy-ldmh2                     1/1     Running   0          23h
kube-proxy-ljb7q                     1/1     Running   0          23h
kube-scheduler-k8s-master            1/1     Running   1          23h
```
At this stage you are good and install containers . The environment is now ready. If you want to try spinnning up VMs in kubernetes, check out the kubevirt and the steps [here](https://ard92.github.io/2022/05/30/kubevirt.html)

# Uninstall k8s
if you need to install kubernetes

1. Tear down the cluster
The below command needs to be run on all master and worker nodes 
```
kubeadm reset 
```
2. Delete all references
The below command needs to be done on all nodes
```
rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/*
```

