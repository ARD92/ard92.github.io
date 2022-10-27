---
layout: post
title: kubernetes debugging
tags: kubernetes
---

## Set alias for long commands
```
alias k=kubectl
```
### Try it out! 
```
root@k8s-master:~/cka_practice/cert# k get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-ubuntu-kv1-mgvvz   1/1     Running   0          18d
virt-launcher-vsrx-sriov-qrfxm   2/2     Running   0          19d
```

## Check which file is consuming the most space 

```
du -h <dir> 2>/dev/null | grep '[0-9\.]\+G'
```
## Delete all Evicted pods
This can be used when there are 100's of evicted pods and you want to delete all of them
```
kubectl get pods | grep Evicted | awk '{print $1}' | xargs kubectl delete pod
```

## Advertise out of a specific interface
Typically k8s picks the interface with default IP when creating the cluster using kubeadm init. One can specify the interface in order to advertise the API.
using the below flag in `kubeadm init` 
```
--apiserver-advertise-addresses=<the eth1 ip addr>
```

Example
```
kubeadm init --apiserver-advertise-address=192.169.1.11 --pod-network-cidr=10.244.0.0/16
```

## Label a node
Sometimes if node is not labeled as master pods may not get scheduled. Or if node selectors are being used and label mismatch/not found. you can label the node as below 

### Find labels
```
root@master:~# kubectl get nodes --show-labels
NAME      STATUS   ROLES           AGE     VERSION   LABELS
master    Ready    control-plane   2m43s   v1.25.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1   Ready    <none>          116s    v1.25.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux
worker2   Ready    <none>          72s     v1.25.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux
```

### Label a node
```
root@master:~# kubectl label nodes master node-role.kubernetes.io/master=
node/master labeled
root@master:~# kubectl label nodes worker1 node-role.kubernetes.io/worker=
node/worker1 labeled
root@master:~# kubectl label nodes worker2 node-role.kubernetes.io/worker=
```

### Verify
```
root@master:~# kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
master    Ready    control-plane,master   4m26s   v1.25.3
worker1   Ready    worker                 3m39s   v1.25.3
worker2   Ready    worker                 2m55s   v1.25.3
```
