---
layout: post
title: JCNR as a cell site router in O-RAN/V-RAN deployments in L2 model
tags: linux kubernetes 
---

## Bring up testbed with VMs 
Follow [here](https://github.com/ARD92/kubevirt-manifests/tree/main/ubuntu_vm) to bring up 2 VMs to test JCNR vrouter for functional testing 

## Install kubernetes
Once the VMS are up. Install kubernetes. Follow the steps defined in this [link](https://ard92.github.io/2022/04/23/setting-up-k8s-in-ubuntu.html)

## Allow scheduling on master node
Each VM behaves as a single node k8s cluster where JCNR runs. In order for pods to be scheduled, we should taint the master node to allow to run workloads.
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
``` 

## Install Helm
```
wget https://get.helm.sh/helm-v3.9.3-linux-amd64.tar.gz
tar -xvzf helm-v3.9.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin
helm version
```

### Verify
```
root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts# helm version
version.BuildInfo{Version:"v3.9.3", GitCommit:"414ff28d4029ae8c8b05d62aa06c7fe3dee2bc58", GitTreeState:"clean", GoVersion:"go1.17.13"}

root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts# helm ls
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
```

## Check Huge pages
Although kubevirt can reserve hugepages at boottime, In case no huges are reserved, the vrouter pod will not start. 

To fix it allocate hugepages

Default is 2Mi allocated.

In case we need to allocate 1Gb of Huge pages then the number of pages needed would be 1024/2 = 512 . Allocate the same to the VM.

```
vim /etc/sysctl.conf

# add the below

vm.nr_hugepages = 512 
```

## Download JCNR
Download JCNR package from Junipers public downlaods page.

## Load images 

### If containerd is used 
```
gunzip jcnr-cni-images.tar.gz
ctr -n=k8s.io image import jcnr-cni-images.tar

gunzip jcnr-vrouter-images.tar.gz
ctr -n=k8s.io image import jcnr-vrouter-images.tar

gunzip syslog-ng-images.tar.gz
ctr -n=k8s.io image import syslog-ng-images.tar
```

#### Verify images on containerd
```
root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/images# crictl images
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
ERRO[0000] unable to determine image API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory"
IMAGE                                                                                           TAG                 IMAGE ID            SIZE
k8s.gcr.io/pause                                                                                3.6                 6270bb605e12e       302kB
registry.k8s.io/coredns/coredns                                                                 v1.9.3              5185b96f0becf       14.8MB
registry.k8s.io/etcd                                                                            3.5.4-0             a8a176a5d5d69       102MB
registry.k8s.io/kube-apiserver                                                                  v1.25.2             97801f8394908       34.2MB
registry.k8s.io/kube-controller-manager                                                         v1.25.2             dbfceb93c69b6       31.3MB
registry.k8s.io/kube-proxy                                                                      v1.25.2             1c7d8c51823b5       20.3MB
registry.k8s.io/kube-scheduler                                                                  v1.25.2             ca0ea1ee3cfd3       15.8MB
registry.k8s.io/pause                                                                           3.8                 4873874c08efc       311kB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/busybox                             latest              16ea53ea7c652       1.46MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-init                       JCNR-22.3-6         68200ff448a66       85.9MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-k8s-applier                JCNR-22.3-6         1b5d1eeae54ea       103MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-k8s-crdloader              JCNR-22.3-6         70c8c60e2eaa4       164MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-k8s-deployer               JCNR-22.3-6         d76a10010512b       82.8MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-telemetry-exporter         JCNR-22.3-6         e4d356dfce274       39.5MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-tools                      JCNR-22.3-6         d7f279459665f       1.21GB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-vrouter-agent              JCNR-22.3-6         a7917a6fe2cac       408MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-vrouter-dpdk               JCNR-22.3-6         0bd00695dbafc       261MB
svl-artifactory.juniper.net/atom-docker/cn2/bazel-build/dev/contrail-vrouter-kernel-init-dpdk   JCNR-22.3-6         8b1686b319002       83MB
svl-artifactory.juniper.net/atom_virtual_docker/busybox                                         latest              7138284460ffa       1.46MB
svl-artifactory.juniper.net/junos-docker-local/warthog/busybox                                  latest              7138284460ffa       1.46MB
svl-artifactory.juniper.net/contrail-docker/syslog-ng                                           v6                  58b90805f1f13       84MB
svl-artifactory.juniper.net/junos-docker-local/warthog/crpd                                     22.3R1.8            1a121bbe53437       468MB
svl-artifactory.juniper.net/junos-docker-local/warthog/crpdconfig-generator                     v3                  c79726fa9a654       99.9MB
svl-artifactory.juniper.net/junos-docker-local/warthog/jcnr-cni                                 20220918-4adf886    ee36612bcdcd1       33MB
root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/images#
```

### If docker is used 
WIP

## Load secrets
### save rootpass as base64
- create a file named `rootPasswordFile` with password as clear string in a single line and save.
- run `base64 rootPasswordFile`
- The above will produce a base64 string. Copy this to secrts/jcnr-secrets.yaml 

### Save license as base64
- similar to above. option the license file for cRPD
- run `base64 license.txt`
- save the generated value to `secrets/jcnr-secrets.yaml`


### Apply license 
```
kubectl apply -f secrets/jcnr-secret.yml
```
## Modify the helm chart 
Modify values.yaml file to match the interface naming convention.

### Fabric interface
This is the interface towards the TOR. 

### Fabric workload interface
This is the interface towards RU unit. Used for management. Can be VF/PF. 

### Workload interfaces
This will be added later once vrouter comes up. 

### Huge pages values
Ensure huge pages is enabled, else vrouter will not come up. modify jcnr/jcnr-vrouter/values.yaml file. Use either 2Mi or Gi form accordingly. By default things would be in 2Mi values.

## Apply helm chart 
once the charts are edited, install the helm chart.

```
helm install jcnr .
```

## If you want to use uio_generic
```
sudo apt install linux-modules-extra-5.4.0-126-generic
```
## Troubleshooting
1. pod doesnt initialize 
    - kubectl describe pods <podname> -n <namespace> 

2. Pods not scheduled. 
    - This could be because the node may not be tainted. Modify them accordingly to allow pod scheduling since this should be an all in one cluster. The master node itself will run pods.

3. If hostname is not "master", label might need to be added 
    ```
    kubectl label node ubuntu node-role.kubernetes.io/master=
    ```
4. if JCNR is using virtio based interfaces. i.e. AIO k8s cluster brought up on KVM VMs using virtio interfaces
    This is a common scenario for testing purposes where SRIOV resources arent available. In such cases ensure the below is applied to VM
    ```
    modprobe 8021q && \
    modprobe vfio-pci && \
    echo Y > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
    ```

5. Install K9s for debugging 
    ```
    curl -sS https://webinstall.dev/k9s | bash
    export PATH="/root/.local/bin:$PATH"
    ```
