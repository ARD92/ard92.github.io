---
layout: post
title: JCNR as a cell site router in O-RAN/V-RAN deployments in L3 model
tags: juniper linux kubernetes
---

## Topology considered 
Below topology is considered for the remainder of the post. 
![topology](/images/jcnr_l3_topo.png)

- JCNR in L3 mode
- multinode k8s cluster 
- master node is not scheduled to run vrouter pods 
- each VM comes up with 4 interfaces
    - pod interface for internet connectivity
    - mgmt interface where all 3 VMs are part of the same subnet
    - Fabric side interface 
    - RU side interface  

## Bring up testbed with VMs 
Follow [here](https://github.com/ARD92/kubevirt-manifests/tree/main/jcnr-l3) to bring up 3 VMs to test JCNR vrouter for functional testing 

## Install kubernetes
Once the VMS are up. Install kubernetes. Follow the steps defined in this [link](https://ard92.github.io/2022/04/23/setting-up-k8s-in-ubuntu.html)
As an alternative you can install in one VM and copy the same disk to other PVs so that it can be faster. This process can be automated as part of CI/CD.

## Iptable rules for forwarding within bridges

By default iptables will not forward packets. Adding the below rule helps in forwarding. 

```
iptables -A FORWARD -i k8s-l3br1 -o k8s-l3br1 -j ACCEPT
```
