---
layout: post
title: JCNR as a cell site router in O-RAN/V-RAN deployments in L2 model
tags: juniper linux kubernetes 
---

## Topology considered 
Below topology is considered for the remainder of the post. 
![topology](/images/jcnr_l2_topo.png)

- each JCNR block represents a VM with k8s installed as an all in one cluster with 3 interfaces (1 mgmt, 1 RU side and 1 towards TOR).
- Both the VMs are connected to a common bridge
- since this aims only at functional testing, virtio interfaces are considered
- SRIOV VFs/PFs can be used if needed 
- Both VMs are spun up on the same compute node and is orchestrated using kubevirt. Refer to the next section for steps to bring up vm

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
When editing the helm chart, ensure you use format correctly. If everthing is specified correctly you will notice the hugepages being used under `more /proc/meminfo | grep Huge`

## Apply helm chart 
once the charts are edited, install the helm chart.

```
helm install jcnr .
```
## Validate all containers
Notice that if vRouter does not start, cRPD also will not start and may be in crashLoopBackoff. This is because a liveness/Readines probe keeps running. So ensure vrouter is UP

```
root@ubuntu:~# kubectl get pods -A
NAMESPACE         NAME                                     READY   STATUS      RESTARTS        AGE
contrail-deploy   contrail-k8s-deployer-797d87c4b9-5vpsb   1/1     Running     1 (10m ago)     16m
contrail          contrail-vrouter-masters-vkr9c           3/3     Running     0               6m59s
default           delete-crpd-dirs-nxm8f                   0/1     Completed   0               22m
default           delete-vrouter-dirs-8mwkv                0/1     Completed   0               22m
kube-flannel      kube-flannel-ds-c2jwr                    1/1     Running     6 (8m27s ago)   2d13h
kube-system       coredns-565d847f94-jrlb5                 1/1     Running     2 (10m ago)     3d12h
kube-system       coredns-565d847f94-rwjzq                 1/1     Running     2 (10m ago)     3d12h
kube-system       etcd-ubuntu                              1/1     Running     4 (9m1s ago)    3d12h
kube-system       kube-apiserver-ubuntu                    1/1     Running     4 (9m1s ago)    3d12h
kube-system       kube-controller-manager-ubuntu           1/1     Running     4 (9m1s ago)    3d12h
kube-system       kube-crpd-worker-ds-mvjbd                1/1     Running     8 (6m56s ago)   16m
kube-system       kube-multus-ds-fj42f                     1/1     Running     3 (9m1s ago)    2d12h
kube-system       kube-proxy-kmhnl                         1/1     Running     4 (9m1s ago)    3d12h
kube-system       kube-scheduler-ubuntu                    1/1     Running     4 (9m1s ago)    3d12h
kube-system       syslog-ng-f68c76988-86fvt                1/1     Running     2 (9m2s ago)    16m
```

## If you want to use uio_generic
```
sudo apt install linux-modules-extra-5.4.0-126-generic
```
## Troubleshooting
1. pod doesnt initialize 
   ```
   kubectl describe pods <podname> -n <namespace> 
   ```

2. Pods not scheduled. 
    - This could be because the node may not be tainted. Modify them accordingly to allow pod scheduling since this should be an all in one cluster. The master node itself will run pods.
    ```
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```
 
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

6. Segmentation fault 
    ```
     kubectl logs contrail-vrouter-masters-7jmch -n contrail

    Defaulted container "contrail-vrouter-agent" out of: contrail-vrouter-agent, contrail-vrouter-agent-dpdk, contrail-vrouter-telemetry-exporter, contrail-init (init), contrail-vrouter-kernel-init-dpdk (init)
    panic: runtime error: invalid memory address or nil pointer dereference
    [signal SIGSEGV: segmentation violation code=0x1 addr=0xa0 pc=0x12e2dc2]

    goroutine 67 [running]:
    main.(*cmdLayer).cmdSignal(0x1fa74f8, 0x0, 0x16d2780, 0x1f263d8, 0x0, 0x0)
    /go/src/ssd-git.juniper.net/contrail/cn2/vrouter-supervisor/main.go:35 +0x22
    main.(*vrouterSupervisor).supervisorProcess.func2(0xc0005a68a0, 0xc0005a2870, 0xc00059c1cb, 0xc00059f440, 0xc0005c6150)
    /go/src/ssd-git.juniper.net/contrail/cn2/vrouter-supervisor/main.go:240 +0x1fc
    created by main.(*vrouterSupervisor).supervisorProcess
    /go/src/ssd-git.juniper.net/contrail/cn2/vrouter-supervisor/main.go:214 +0x51a
    ```

    This could be because of incorrect values.yaml. In the above case, it was because the bond interface configs were commented. solution is to comment the values but not the key

    example of correct way to not use bond interface
    
    ```
        bondInterfaceConfigs:
    #  - name: "bond0"
    #    mode: 1             # ACTIVE_BACKUP MODE
    #    slaveInterfaces:
    #    - "ens1f1"
    ```

7. uio_pci_generic module missing
 
    ```
    root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts/jcnr/charts/jcnr-vrouter# modprobe uio_pci_generic
    modprobe: FATAL: Module uio_pci_generic not found in directory /lib/modules/5.4.0-126-generic
    root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts/jcnr/charts/jcnr-vrouter# apt install linux-modules-extra-5.4.0-126-generic
    ``` 
    you would have to install 
    ```
    apt install linux-modules-extra-5.4.0-126-generic
    sudo modprobe uio_pci_generic
    ```

    verify
    ```
    root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts/jcnr/charts/jcnr-vrouter# lsmod | grep uio_pci_generic
    uio_pci_generic        16384  0
    uio                    20480  1 uio_pci_generic
    ```

8. Verify vfio is loaded
    Ubuntu 20.04 is already loaded with vfio modules. you can validate by 
    ```
    root@ubuntu:~/Juniper_Cloud_Native_Router_22.3/helm_charts/jcnr/charts/jcnr-vrouter# cat /lib/modules/$(uname -r)/modules.builtin | grep vfio
    kernel/drivers/vfio/vfio.ko
    kernel/drivers/vfio/vfio_virqfd.ko
    kernel/drivers/vfio/vfio_iommu_type1.ko
    kernel/drivers/vfio/pci/vfio-pci.ko
    ```

9. vrouter continuously crashes with illegal instruction 
    ```
    time="2022-10-10T14:37:29Z" level=info msg="vrouter dpdk process exited"
    time="2022-10-10T14:37:29Z" level=error msg="vrouter dpdk process exit response: signal: illegal instruction"
    time="2022-10-10T14:37:29Z" level=info msg="start restoring l2 interfaces"
    ```
    Ensure the avx2 instruction set is enabled on the compute. Without this vrouter will keep crashing. you can check the cpu flags on `lscpu`

    Example
    ```
    root@k8s-worker2:~# lscpu
    Architecture:                    x86_64
    CPU op-mode(s):                  32-bit, 64-bit
    Byte Order:                      Little Endian
    Address sizes:                   46 bits physical, 48 bits virtual
    CPU(s):                          48
    On-line CPU(s) list:             0-47
    Thread(s) per core:              2
    Core(s) per socket:              12
    Socket(s):                       2
    NUMA node(s):                    2
    Vendor ID:                       GenuineIntel
    CPU family:                      6
    Model:                           63
    Model name:                      Intel(R) Xeon(R) CPU E5-2690 v3 @ 2.60GHz
    Stepping:                        2
    CPU MHz:                         3255.867
    CPU max MHz:                     3500.0000
    CPU min MHz:                     1200.0000
    BogoMIPS:                        5194.11
    Virtualization:                  VT-x
    L1d cache:                       768 KiB
    L1i cache:                       768 KiB
    L2 cache:                        6 MiB
    L3 cache:                        60 MiB
    NUMA node0 CPU(s):               0-11,24-35
    NUMA node1 CPU(s):               12-23,36-47
    Vulnerability Itlb multihit:     KVM: Mitigation: Split huge pages
    Vulnerability L1tf:              Mitigation; PTE Inversion; VMX conditional cache flushes, SMT vulnerable
    Vulnerability Mds:               Mitigation; Clear CPU buffers; SMT vulnerable
    Vulnerability Meltdown:          Mitigation; PTI
    Vulnerability Spec store bypass: Mitigation; Speculative Store Bypass disabled via prctl and seccomp
    Vulnerability Spectre v1:        Mitigation; usercopy/swapgs barriers and __user pointer sanitization
    Vulnerability Spectre v2:        Mitigation; Retpolines, IBPB conditional, IBRS_FW, STIBP conditional, RSB filling
    Vulnerability Srbds:             Not affected
    Vulnerability Tsx async abort:   Not affected
    Flags:                           fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc c
                                        puid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm cpuid_fault epb invpcid
                                         _single pti intel_ppin ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm ida arat pln pts md_clear flush_l1d
   ```

10. Dpdk pod crashes with the error
    ```
    05:06:07,478 EAL: VFIO support initialized                                 
   2 05:06:07,647 EAL: eal_memalloc_alloc_seg_bulk(): couldn't find suitable me
   2 05:06:07,647 EAL: Cannot init memory                                      
   2 05:06:07,648 VROUTER: Error initializing EAL                              
   2 05:06:07,648 VROUTER: Error DPDK initialization failed
   ```
   In this case uio_pci_generic module would be missing. Ensure that it is loaded. Refer to troubleshooting steps #7 

11. MTU error and VIFs not created even though vrouter and crpd  pods is up and running
    ```
    tail -100 /var/log/jcnr/contrail-vrouter-dpdk.log

    2022-10-16 14:59:14,975 VROUTER: Using 1 TX queues, 1 RX queues
    2022-10-16 14:59:14,975 MTU (9198) > device max MTU (1500) for port_id 1
    2022-10-16 14:59:14,975 VROUTER:     error configuring eth dev 1: Invalid argument (22)
    2022-10-16 14:59:14,975 DPCORE: vr_purel2_bd_cleanup: Clearing bd table for vif:2
    2022-10-16 14:59:14,975 VROUTER: Deleting vif 2 eth device
    2022-10-16 14:59:14,975 VROUTER:     error deleting eth dev: already removed
    2022-10-16 14:59:20,606 VROUTER: Adding vif 2 (gen. 4) eth device 1 PCI 0000:03:00.0 MAC aa:aa:2c:d3:af:b7 (vif MAC aa:aa:2c:d3:af:b7)
    2022-10-16 14:59:20,606 MTU (9000) > device max MTU (1500) for port_id 1
    2022-10-16 14:59:20,606 VROUTER:     error adding eth dev enp3s0: already added
    2022-10-16 14:59:20,606 DPCORE: vr_purel2_bd_cleanup: Clearing bd table for vif:2
    2022-10-16 14:59:20,606 VROUTER: Deleting vif 2 eth device
    2022-10-16 14:59:20,606 VROUTER:     error deleting eth dev: already removed
    2022-10-16 14:59:20,607 VROUTER: Adding vif 1 (gen. 5) eth device 0 PCI 0000:02:00.0 MAC 26:e6:bf:4e:8f:9c (vif MAC 26:e6:bf:4e:8f:9c)
    2022-10-16 14:59:20,607 MTU (9000) > device max MTU (1500) for port_id 0
    2022-10-16 14:59:20,607 VROUTER:     error adding eth dev enp2s0: already added
    2022-10-16 14:59:20,607 DPCORE: vr_purel2_bd_cleanup: Clearing bd table for vif:1
    2022-10-16 14:59:20,607 VROUTER: Deleting vif 1 eth device
    2022-10-16 14:59:20,607 VROUTER:     error deleting eth dev: already removed
    2022-10-16 14:59:20,607 DPCORE: vr_purel2_bd_add_req: vif not found with index:1
    ``` 
    Change the MTU following cni documentation. in this case, i used [bridge-cni](https://www.cni.dev/plugins/current/main/bridge/#example-l2-only-configuration)

    ```
    example of kubevirt vm Nad

    root@k8s-master:/home/aprabh/kubevirt/ubuntu_vm# more NAD/vm2/nad-intf1.yaml
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
    name: intf1
    spec:
        config: '{
        "cniVersion": "0.3.0",
        "name": "intf1",
        "type": "bridge",
        "bridge": "br1",
        "isGateway": true,
        "mtu": 9198,
        "ipam": {
        "type": "host-local",
        "subnet": "192.168.6.0/24"
        }
    }'
    ```

## Run a dpdk pod to test traffic 
We can use dpdk pktgen to test out traffic across the JCNR VMs. 

### Create the NAD 
```
root@ubuntu:~# more nad_bd.yml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vswitch-trunk1
spec:
  config: '{
    "cniVersion":"0.4.0",
    "name": "vswitch-trunk1",
    "type": "jcnr",
    "args": {
      "instanceName": "vswitch",
      "instanceType": "virtual-switch",
      "dataplane":"dpdk",
      "vlanIdList": "100"
    },
    "kubeConfig":"/etc/kubernetes/kubelet.conf"
  }'
```

Apply the NAD
```
kubectl apply -f nad_bd.yml
```

### Create the pod yaml
```
root@ubuntu:~# more pktgen_nad.yml
apiVersion: v1
kind: Pod
metadata:
  name:   odu-trunk-1
  annotations:
    k8s.v1.cni.cncf.io/networks: vswitch-trunk1
spec:
  containers:
    - name: odu-trunk
      image: svl-artifactory.juniper.net/junos-docker-local/warthog/pktgen19116:20210303
      imagePullPolicy: IfNotPresent
      securityContext:
        privileged: true
      resources:
        requests:
          memory: 1Gi
        limits:
          hugepages-2Mi: 1024Mi
      env:
        - name: KUBERNETES_POD_UID
          valueFrom:
            fieldRef:
               fieldPath: metadata.uid
      volumeMounts:
        - name: dpdk
          mountPath: /dpdk
          subPathExpr: $(KUBERNETES_POD_UID)
        - mountPath: /dev/hugepages
          name: hugepage
  volumes:
    - name: dpdk
      hostPath:
        path: /var/run/jcnr/containers
    - name: hugepage
      emptyDir:
        medium: HugePages
```

Apply the pod

```
kubectl apply -f pktgen_nad.yml
```

## Test traffic using pktgen

### Get into odupod shell
```
kubectl exec -it <odupod> sh
```
### Find out sock and mac address
```
sh-4.2# more /dpdk/dpdk-interfaces.json
[
    {
        "name": "net1",
        "vhost-adaptor-path": "/dpdk/vhost-net1.sock",
        "vhost-adaptor-mode": "client",
        "mac-address": "02:00:00:F1:91:62"
    }
]
```
Ensure you use the same MAC as source MAC and far end pod as destination MAC. The vhost-adaptor-path has to be used when starting the pktgen app. 

### Edit the pktgen.pkt file 
used vlan 100 to send traffic with a vlan id 
```
sh-4.2# more pktgen.pkt

set 0 src mac 02:00:00:F1:91:62
set 0 src ip 20.1.1.2/24
set 0 dst mac 02:00:00:8D:66:74
set 0 dst ip 20.1.1.1/24
set 0 burst 32
set 0 rate 1
set 0 size 64
enable 0 vlan
set 0 vlan 100
```

### Start pktgen app
Notice that the mac address used is same from whats obtained in the previous sections.

```
rm -rf /dpdk/vhost-net1.sock

/usr/local/bin/pktgen  -l 4-5 -n 4 --socket-mem 1024 --no-pci --file-prefix=pktgen --vdev=net_virtio_user1,mac=02:00:00:8D:66:74,path=/dpdk/vhost-net1.sock,server=1 --single-file-segments -- -P -m 5.0 -f pktgen.pkt
/usr/local/bin/pktgen  -l 4-5 -n 4 --socket-mem 1024 --no-pci --file-prefix=pktgen --vdev=net_virtio_user1,mac=02:00:00:F1:91:62,path=/dpdk/vhost-net1.sock,server=1 --single-file-segments -- -P -m 5.0 -f pktgen.pkt
```

### Run 
```
type “start 0” to start traffic and “stop 0” to stop traffic.

/ Ports 0-0 of 1   <Main Page>  Copyright (c) <2010-2020>, Intel Corporation
  Flags:Port        : P------Single  VLAN:0
Link State          :         <UP-10000-FD>      ---Total Rate---
Pkts/s Max/Rx       :         149312/148042         149312/148042
       Max/Tx       :         150208/148576         150208/148576
MBits/s Rx/Tx       :                 99/99                 99/99
Broadcast           :                     0
Multicast           :                     0
Sizes 64            :               8428099
      65-127        :                     0
      128-255       :                     0
      256-511       :                     0
      512-1023      :                     0
      1024-1518     :                     0
Runts/Jumbos        :                   0/0
ARP/ICMP Pkts       :                   0/0
Errors Rx/Tx        :                   0/0
Total Rx Pkts       :               8328195
      Tx Pkts       :              60407360
      Rx MBs        :                  5596
      Tx MBs        :                 40593
                    :
Pattern Type        :               abcd...
Tx Count/% Rate     :           Forever /1%
Pkt Size/Tx Burst   :             64 /   32
TTL/Port Src/Dest   :        64/ 1234/ 5678
Pkt Type:VLAN ID    :       IPv4 / TCP:0064
802.1p CoS/DSCP/IPP :             0/  0/  0
VxLAN Flg/Grp/vid   :      0000/    0/    0
IP  Destination     :              20.1.1.2
    Source          :           20.1.1.1/24
MAC Destination     :     02:00:00:f1:91:62
    Source          :     02:00:00:8d:66:74
PCI Vendor/Addr     :     0000:0000/00:00.0
```
Ensure dest pod also generates traffic else it will be flooded. Once traffic is seen from reverse direction, you will notice TX/RX and no broadcast pkts. 

### Validate on bridge 
Since bridge is used on this setup for functional testing we can snoop using tcpdump on the hose where the kubevirt vms were created

```
root@k8s-worker2:~# brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.de28488cab83	no
br1		8000.52525c27df40	no		veth2d90f2c7
							veth6deb051c
br2		8000.d2bb6c20051f	no		veth37a5c307
br3		8000.32460d99cc89	no		veth51786a5e


root@k8s-worker2:~# tcpdump -nei br1
21:19:11.652706 02:00:00:f1:91:62 > 02:00:00:8d:66:74, ethertype 802.1Q (0x8100), length 60: vlan 100, p 0, ethertype IPv4, 20.1.1.2.1234 > 20.1.1.1.5678: Flags [.], seq 0:2, ack 1, win 8192, length 2
21:19:11.652708 02:00:00:8d:66:74 > 02:00:00:f1:91:62, ethertype 802.1Q (0x8100), length 60: vlan 100, p 0, ethertype IPv4, 20.1.1.1.1234 > 20.1.1.2.5678: Flags [.], seq 0:2, ack 1, win 8192, length 2
21:19:11.652707 02:00:00:f1:91:62 > 02:00:00:8d:66:74, ethertype 802.1Q (0x8100), length 60: vlan 100, p 0, ethertype IPv4, 20.1.1.2.1234 > 20.1.1.1.5678: Flags [.], seq 0:2, ack 1, win 8192, length 2
21:19:11.652709 02:00:00:8d:66:74 > 02:00:00:f1:91:62, ethertype 802.1Q (0x8100), length 60: vlan 100, p 0, ethertype IPv4, 20.1.1.1.1234 > 20.1.1.2.5678: Flags [.], seq 0:2, ack 1, win 8192, length 2
21:19:11.652708 02:00:00:f1:91:62 > 02:00:00:8d:66:74, ethertype 802.1Q (0x8100), length 60: vlan 100, p 0, ethertype IPv4, 20.1.1.2.1234 > 20.1.1.1.5678: Flags [.], seq 0:2, ack 1, win 8192, length 2
< ----- snipped ----- >
```

## Verifying vrouter traffic handling

1. Ensure vrouter VIFs are created
    ```
    root@ubuntu:~# kubectl exec -it contrail-vrouter-masters-s7m6m -n contrail sh
    kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
    Defaulted container "contrail-vrouter-agent" out of: contrail-vrouter-agent, contrail-vrouter-agent-dpdk, contrail-vrouter-telemetry-exporter, contrail-init (init), contrail-vrouter-kernel-init-dpdk (init)
    # vif --list
    Vrouter Operation Mode: PureL2
    Vrouter Interface Table
    
    Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
           Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
           D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
           Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
           Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
           Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
           HbsR=HBS Right Intf, Ig=Igmp Trap Enabled, Ml=MAC-IP Learning Enabled, Me=Multicast Enabled
    
    vif0/0      Socket: unix
                Type:Agent HWaddr:00:00:5e:00:01:00
                Vrf:65535 Flags:L2 QOS:-1 Ref:3
                RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0
                RX packets:0  bytes:0 errors:0
                TX packets:2071  bytes:215384 errors:0
                Drops:0
    
    vif0/1      PCI: 0000:02:00.0
                Type:Physical HWaddr:e6:53:a9:ac:30:45
                Vrf:65535 Flags:L2Vof QOS:-1 Ref:7
                RX queue  packets:177510037 errors:40999
                RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 40999
                Fabric Interface: 0000:02:00.0  Status: UP  Driver: net_virtio
                Vlan Mode: Trunk  Vlan: 100 1000-1007
                RX packets:177510027  bytes:10650601620 errors:11
                TX packets:60166656  bytes:3609999360 errors:0
                Drops:109246546
    
    vif0/2      PCI: 0000:03:00.0
                Type:Workload HWaddr:a6:7d:c3:ff:81:0e
                Vrf:0 Flags:L2Vof QOS:-1 Ref:7
                RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0
                Fabric Interface: 0000:03:00.0  Status: UP  Driver: net_virtio
                Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
                RX packets:0  bytes:0 errors:0
                TX packets:109203965  bytes:6115422040 errors:0
                Drops:0
    
    vif0/3      PMD: vhostnet1-79300016-c2af-4d05-94
                Type:Virtual HWaddr:02:00:00:f1:91:62
                Vrf:65535 Flags:L2 QOS:-1 Ref:9
                RX port   packets:60176672 errors:0
                RX queue  packets:60168416 errors:8256
                RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 8256
                Vlan Mode: Trunk  Vlan: 100
                RX packets:60168416  bytes:3610104960 errors:0
                TX packets:177511819  bytes:10650709140 errors:0
                Drops:8756
                TX port   packets:112247966 errors:65263853
    ```

2. Once traffic is running we can check mac address on vrouter and crpd 
    From vrouter
    ```
    # purel2cli --mac show
    ==================================================
    ||    MAC           vlan       port    hit_count||
    ==================================================
    02:00:00:8d:66:74 100        1          190833409
    02:00:00:f1:91:62 100        3          50383437
    ```

    from crpd 
    ```
    root@ubuntu> show bridge mac-table
    
    MAC flags         (S - Static MAC, D - Dynamic MAC)
    Routing Instance : default-domain:default-project:ip-fabric:__default__
    Bridging domain VLAN id : 100
    MAC                  MAC                Logical
    address              flags              interface
    
    02:00:00:8d:66:74      D                 enp2s0
    02:00:00:f1:91:62      D                 vhostnet1-79300016-c2af-4d05-94
    
    
    root@ubuntu> show bridge statistics
    
    Bridge domain vlan-id: 100
       Local interface: enp2s0
          Broadcast packets Tx  : 0           Rx  : 0
          Multicast packets Tx  : 0           Rx  : 0
          Unicast packets Tx    : 101593656   Rx  : 218462929
          Broadcast bytes Tx    : 0           Rx  : 0
          Multicast bytes Tx    : 0           Rx  : 0
          Unicast bytes Tx      : 6095619360  Rx  : 13107775740
          Flooded packets       : 0
          Flooded bytes         : 0
       Local interface: enp3s0
          Broadcast packets Tx  : 0           Rx  : 0
          Multicast packets Tx  : 0           Rx  : 0
          Unicast packets Tx    : 0           Rx  : 0
          Broadcast bytes Tx    : 0           Rx  : 0
          Multicast bytes Tx    : 0           Rx  : 0
          Unicast bytes Tx      : 0           Rx  : 0
          Flooded packets       : 109203965
          Flooded bytes         : 6552237900
       Local interface: vhostnet1-79300016-c2af-4d05-94
          Broadcast packets Tx  : 0           Rx  : 0
          Multicast packets Tx  : 0           Rx  : 0
          Unicast packets Tx    : 109258964   Rx  : 101593664
          Broadcast bytes Tx    : 0           Rx  : 0
          Multicast bytes Tx    : 0           Rx  : 0
          Unicast bytes Tx      : 6555537840  Rx  : 6095619840
          Flooded packets       : 109203965
          Flooded bytes         : 6552237900
    ```

3. Check traffic stats
    ```
    vif --list --rate

    Interface rate statistics
    -------------------------
    Interface name                VIF ID                        RX                            TX
    Agent: unix                   vif0/0                  0        0                    0        0
    Physical: 0000:02:00.0        vif0/1                  0        146352               0        147263
    Workload: 0000:03:00.0        vif0/2                  0        0                    0        0
    Virtual: vhostnet1-79300016-c2af-4d05-94vif0/3                  0        147263               0        146320
    ```
