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
