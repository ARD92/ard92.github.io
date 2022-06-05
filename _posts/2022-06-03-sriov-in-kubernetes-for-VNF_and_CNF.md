---
layout: post
title: SR-IOV interfaces for VNF and CNF in kubernetes 
tags: linux kubernetes
---

# Setting up SRIOV based VNF and CNF in kubernetes

Kubernetes allows one to create CNF and VNF (using kubervirt. see [here](https://ard92.github.io/2022/05/30/kubevirt.html)) to better life cycle manage them. There are always requirements for running high performing cNF and vNF for which there is a need for using SR-IOV and DPDK. In this post we will see how SR-IOV CNI can be used to instantiate pods/VMs to use a VF and process traffic. 

## Enabling SRIOV 

Single root input out virtualization allows one for higher network throughput by bypassing the kernel. It shows a PCIe device to appear as multiple separate physical PCIe devices. Further these physical functions (PF) can be broken into multiple virtual functions (VF) and allocated to a VNF/CNF directly, effecting sending traffic directly into the respective VNF/CNF. There are tons of documents explaining what is SRIOV so we would be skipping that :)  

In order to enable SRIOV on a compute, below are to be done 
1. Enable VT-d, SRIOV on BIOS
    This differs from manufacturer. check for the material on respective website on how to enable on BIOS
2. Modify Grub file to support 
    
### Modifying grub

In order to make this change, edit the file `/etc/default/grub` with the below contents
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=10 isolcpu=2-23"
```

#### Update grub and reboot 
```
update-grub
```
reboot the node at this point

#### Validate the change 
```
root@k8s-master:~/vsrx_sriov_fix# virt-host-validate
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
  QEMU: Checking for cgroup 'cpu' controller support                         : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
  QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
  QEMU: Checking for cgroup 'memory' controller support                      : PASS
  QEMU: Checking for cgroup 'devices' controller support                     : PASS
  QEMU: Checking for cgroup 'blkio' controller support                       : PASS
  QEMU: Checking for device assignment IOMMU support                         : PASS
  QEMU: Checking if IOMMU is enabled by kernel                               : PASS
  QEMU: Checking for secure guest support                                    : WARN (Unknown if this platform has Secure Guest support)
   LXC: Checking for Linux >= 2.6.26                                         : PASS
   LXC: Checking for namespace ipc                                           : PASS
   LXC: Checking for namespace mnt                                           : PASS
   LXC: Checking for namespace pid                                           : PASS
   LXC: Checking for namespace uts                                           : PASS
   LXC: Checking for namespace net                                           : PASS
   LXC: Checking for namespace user                                          : PASS
   LXC: Checking for cgroup 'cpu' controller support                         : PASS
   LXC: Checking for cgroup 'cpuacct' controller support                     : PASS
   LXC: Checking for cgroup 'cpuset' controller support                      : PASS
   LXC: Checking for cgroup 'memory' controller support                      : PASS
   LXC: Checking for cgroup 'devices' controller support                     : PASS
   LXC: Checking for cgroup 'freezer' controller support                     : PASS
   LXC: Checking for cgroup 'blkio' controller support                       : PASS
   LXC: Checking if device /sys/fs/fuse/connections exists                   : PASS
```
### Check hostlevel params

```
root@k8s-master:~/vsrx_sriov_fix# lscpu
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   46 bits physical, 48 bits virtual
CPU(s):                          24
On-line CPU(s) list:             0-23
Thread(s) per core:              1
Core(s) per socket:              12
Socket(s):                       2
NUMA node(s):                    2
Vendor ID:                       GenuineIntel
CPU family:                      6
Model:                           63
Model name:                      Intel(R) Xeon(R) CPU E5-2690 v3 @ 2.60GHz
Stepping:                        2
CPU MHz:                         1730.345
BogoMIPS:                        5193.46
Virtualization:                  VT-x
L1d cache:                       768 KiB
L1i cache:                       768 KiB
L2 cache:                        6 MiB
L3 cache:                        60 MiB
NUMA node0 CPU(s):               0-5,12-17
NUMA node1 CPU(s):               6-11,18-23
Vulnerability Itlb multihit:     KVM: Mitigation: Split huge pages
Vulnerability L1tf:              Mitigation; PTE Inversion; VMX conditional cache flushes, SMT disabled
Vulnerability Mds:               Mitigation; Clear CPU buffers; SMT disabled
Vulnerability Meltdown:          Mitigation; PTI
Vulnerability Spec store bypass: Mitigation; Speculative Store Bypass disabled via prctl and seccomp
Vulnerability Spectre v1:        Mitigation; usercopy/swapgs barriers and __user pointer sanitization
Vulnerability Spectre v2:        Mitigation; Retpolines, IBPB conditional, IBRS_FW, RSB filling
Vulnerability Srbds:             Not affected
Vulnerability Tsx async abort:   Not affected
Flags:                           fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse
                                  sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtop
                                 ology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma
                                  cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdra
                                 nd lahf_lm abm cpuid_fault epb invpcid_single pti intel_ppin ssbd ibrs ibpb stibp tpr_shadow vnmi fle
                                 xpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc c
                                 qm_occup_llc dtherm arat pln pts md_clear flush_l1d
```
Notice that `VT-x` is enabled for virtualization.

#### Identifying NICs 

Identify which NICs support SRIOV. In my example, i have Intel 82599 NICs 

```
root@k8s-master:~/vsrx_sriov_fix# lspci | grep Ethernet
02:00.0 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (rev 01)
02:00.1 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (rev 01)
02:00.2 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (rev 01)
02:00.3 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (rev 01)
81:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
81:00.1 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)


lspci -nn | grep 82599

04:00.0 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
04:00.1 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
04:10.1 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.3 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.5 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.7 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.1 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.3 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.5 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.7 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)

```
#### Gather Bus info
```
root@k8s-master:~/vsrx_sriov_fix# lshw -c network -businfo
Bus info          Device        Class          Description
==========================================================
pci@0000:02:00.0  eno1          network        NetXtreme BCM5719 Gigabit Ethernet PCIe
pci@0000:02:00.1  eno2          network        NetXtreme BCM5719 Gigabit Ethernet PCIe
pci@0000:02:00.2  eno3          network        NetXtreme BCM5719 Gigabit Ethernet PCIe
pci@0000:02:00.3  eno4          network        NetXtreme BCM5719 Gigabit Ethernet PCIe
pci@0000:81:00.0  ens6f0        network        82599ES 10-Gigabit SFI/SFP+ Network Connection
pci@0000:81:00.1  ens6f1        network        82599ES 10-Gigabit SFI/SFP+ Network Connection
pci@0000:84:00.0  ens5f0        network        82599ES 10-Gigabit SFI/SFP+ Network Connection
pci@0000:84:00.1  ens5f1        network        82599ES 10-Gigabit SFI/SFP+ Network Connection
                  flannel.1     network        Ethernet interface
                  veth52b3d68c  network        Ethernet interface
                  veth16386eb7  network        Ethernet interface
                  veth24fe53cb  network        Ethernet interface
```

#### Validate drivers using Ethtool

```
root@k8s-master:~/vsrx_sriov_fix# ethtool -i ens6f0
driver: ixgbe
version: 5.1.0-k
firmware-version: 0x80000835, 1.2688.0
expansion-rom-version:
bus-info: 0000:81:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes
```
### Create SR-IOV interfaces 
```
echo 8 > /sys/class/net/ens6f0/device/sriov_numvfs
```
In this example, it creates 8 VFs for the device `ens6f0`. 

#### Identify VFs 
```
root@k8s-master:~/vsrx_sriov_fix# ip link show ens6f0
5: ens6f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 8c:dc:d4:a9:c3:2c brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether 82:e6:28:8d:b3:09 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 1     link/ether ae:4b:e8:d8:75:66 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 2     link/ether b6:a9:9e:be:5f:e1 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 3     link/ether a6:b5:ac:73:47:2b brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 4     link/ether aa:f1:42:81:1e:7c brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 5     link/ether c2:7f:4f:0d:4f:ca brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 6     link/ether 1e:fa:b9:05:0c:08 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 7     link/ether b2:84:1b:df:2c:e4 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
```
Notice that 8 VFs are created .

#### Add vlan tags so that the VFs can be differentiated 
```
ip link set dev ens6f1 vf 0 vlan 1000
ip link set dev ens6f1 vf 1 vlan 1001
ip link set dev ens6f1 vf 2 vlan 1002
ip link set dev ens6f1 vf 3 vlan 1003
ip link set dev ens6f1 vf 4 vlan 1004
ip link set dev ens6f1 vf 5 vlan 1005
ip link set dev ens6f1 vf 6 vlan 1006
ip link set dev ens6f1 vf 7 vlan 1007
```

The `lspci | grep Ethernet` also provides information on the VF

## Setup SRIOV device plugin 

In order to provision SRIOV based vNF/cNF, we need 2 parts 
- multus (to help use multiple CNIs and interfaces)
- CNI cpaable of consuming the network device allocated.
    - SR-IOV CNI (for SR-IOV VFs)
    - Host device CNI (for PCI PF)

For cNF, things are a little straight forward. an SRIOV PF/VF can be inserted. However for VM using Kubevirt, kubevirt relies of userspace VFIO drivers. Hence ensure to use `sriov_dpdk` . The steps have been outlined in the following sections.

For the NIC support look at link [here](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)

Create the SR-IOV VFs before proceeding to the below steps. They have been covered in the above sections.

## Install SRIOV device plugin

```
docker pull ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:latest
git clone https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin.git
make image
```
### Edit the configMap. 
In my example, since i am using 82599 NIC, below is how the config map looks like
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [{
                "resourceName": "intel_sriov_netdevice",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed", "1889"],
                    "drivers": ["i40evf", "iavf", "ixgbevf"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["10ed"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["ens6f1","ens1f1", "ens3f1"]
                }
            }
        ]
    }
```

Note that `intel_sriov_dpdk` stanza with pf names have been mentioned. The PF names are the ones where VF were created earlier. These will get advertised over the device plugin so that multus can pick up the device IDs to schedule pods/VMs as secondary intefaces.

### Apply the deployment 
```
kubectl create -f deployments/configMap.yaml
kubectl create -f deployments/k8s-v1.16/sriovdp-daemonset.yaml
```

### Validate 
The device plugin will run on all the nodes(master and worker). 
```
root@k8s-master:~/sriov_cni# kubectl get pods -A | grep sriov
kube-system   kube-sriov-device-plugin-amd64-7pxsl   1/1     Running   0              141m
kube-system   kube-sriov-device-plugin-amd64-8dhrc   1/1     Running   0              141m
kube-system   kube-sriov-device-plugin-amd64-dg7zg   1/1     Running   0              122m
```

## Install SRIOV CNI
This works with the device plugin for VF allocation. 

```
git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
kubectl apply -f images/k8s-v1.16/sriov-cni-daemonset.yaml
```

More information available [here](https://github.com/k8snetworkplumbingwg/sriov-cni)

### Validate 

#### Validate cni binary 
validate sriov cni now present on all nodes under `/opt/cni/bin`

```
root@k8s-master:~/sriov_cni# ls -l /opt/cni/bin/ | grep sriov
-rwxr-xr-x 1 root root  3687719 Jun  4 16:59 sriov
```

#### Validate pods 
```
root@k8s-master:~/sriov_cni# kubectl get pods -A | grep sriov-cni
kube-system   kube-sriov-cni-ds-amd64-92qn5          1/1     Running   0              107m
kube-system   kube-sriov-cni-ds-amd64-f9446          1/1     Running   0              107m
kube-system   kube-sriov-cni-ds-amd64-xhtfj          1/1     Running   0              107m
```

## Bind interfaces to userspace VFIO so that VMs can use it. 

If this step is not done, kubevirt will throw the below error and the VM will never start since the VF interface attached will not allow it to start.

```
Message: server error. command SyncVMI failed: "LibvirtError(Code=67, Domain=10, Message='unsupported configuration: host doesn't support passthrough of host PCI devices')"
```
This is because, kubevirt relies on VFIO drivers and we need to attach as a userspace interface.

### Bind using dpdk tools

#### Download the tool 
```
git clone https://github.com/DPDK/dpdk.git
cd dpdk/usertools/
```
#### Display the devices that can be bound 
```
./dpdk-devbind.py -s


Network devices using kernel driver
===================================
0000:03:00.0 'NetXtreme BCM5719 Gigabit Ethernet PCIe 1657' if=eno1 drv=tg3 unused=vfio-pci *Active*
0000:03:00.1 'NetXtreme BCM5719 Gigabit Ethernet PCIe 1657' if=eno2 drv=tg3 unused=vfio-pci
0000:03:00.2 'NetXtreme BCM5719 Gigabit Ethernet PCIe 1657' if=eno3 drv=tg3 unused=vfio-pci
0000:03:00.3 'NetXtreme BCM5719 Gigabit Ethernet PCIe 1657' if=eno4 drv=tg3 unused=vfio-pci
0000:04:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens1f0 drv=ixgbe unused=vfio-pci
0000:04:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens1f1 drv=ixgbe unused=vfio-pci *Active*
0000:07:00.0 '82580 Gigabit Network Connection 150e' if=ens2f0 drv=igb unused=vfio-pci
0000:07:00.1 '82580 Gigabit Network Connection 150e' if=ens2f1 drv=igb unused=vfio-pci
0000:07:00.2 '82580 Gigabit Network Connection 150e' if=ens2f2 drv=igb unused=vfio-pci
0000:07:00.3 '82580 Gigabit Network Connection 150e' if=ens2f3 drv=igb unused=vfio-pci
0000:04:10.1 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:10.3 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:10.5 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:10.7 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:11.1 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:11.3 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:11.5 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
0000:04:11.7 '82599 Ethernet Controller Virtual Function 10ed' drv=vfio-pci unused=ixgbevf
```

#### Unbind the interface and bind to the vfio driver 
```
./dpdk-devbind.py -u 04:10.1
./dpdk-devbind.py -u 04:10.3
./dpdk-devbind.py -u 04:10.5
./dpdk-devbind.py -u 04:10.7
./dpdk-devbind.py -u 04:11.1
./dpdk-devbind.py -u 04:11.3
./dpdk-devbind.py -u 04:11.5
./dpdk-devbind.py -u 04:11.7

./dpdk-devbind.py -b vfio-pci 04:10.1
./dpdk-devbind.py -b vfio-pci 04:10.3
./dpdk-devbind.py -b vfio-pci 04:10.5
./dpdk-devbind.py -b vfio-pci 04:10.7
./dpdk-devbind.py -b vfio-pci 04:11.1
./dpdk-devbind.py -b vfio-pci 04:11.3
./dpdk-devbind.py -b vfio-pci 04:11.5
./dpdk-devbind.py -b vfio-pci 04:11.7
```

## Validate annotations 
Now that all the necessary CNI and plugins are installed, we can try starting a Pod with SRIOV interface . Verify if the nodes are advertising the sriov capabilities. 
else, you may need to look at logs of the `sriov-cni` and `sriov-device-plugin` deployments/pods.

```
root@k8s-master:~/sriov_cni# kubectl get nodes
NAME          STATUS   ROLES                  AGE    VERSION
k8s-master    Ready    control-plane,master   3d8h   v1.23.6
k8s-worker1   Ready    <none>                 3d8h   v1.23.6
k8s-worker2   Ready    <none>                 3d8h   v1.23.6
```


```
root@k8s-master:~/sriov_cni# kubectl describe node k8s-worker1
Name:               k8s-worker1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
< ------ snipped ------ >
Allocatable:
  cpu:                              40
  devices.kubevirt.io/kvm:          1k
  devices.kubevirt.io/sev:          0
  devices.kubevirt.io/tun:          1k
  devices.kubevirt.io/vhost-net:    1k
  ephemeral-storage:                795499682523
  hugepages-1Gi:                    10Gi
  intel.com/intel_sriov_dpdk:       8
  intel.com/intel_sriov_netdevice:  0

< ------------ snipped ------- >

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                         Requests     Limits
  --------                         --------     ------
  cpu                              650m (1%)    1300m (3%)
  memory                           1920Mi (0%)  350Mi (0%)
  ephemeral-storage                0 (0%)       0 (0%)
  hugepages-1Gi                    0 (0%)       0 (0%)
  devices.kubevirt.io/kvm          0            0
  devices.kubevirt.io/sev          0            0
  devices.kubevirt.io/tun          0            0
  devices.kubevirt.io/vhost-net    0            0
  intel.com/intel_sriov_dpdk       0            0
  intel.com/intel_sriov_netdevice  0            0
Events:                            <none>
```
Notice that `intel.com/intel_sriov_dpdk` and `intel.com/intel_sriov_netdevice` shows up as advertised.

## Create a Network attachment Definition

```
root@k8s-master:~/vsrx_sriov_fix# more sriov_nad_fxp.yml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriovnet-fxp
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_netdevice
spec:
  config: '{
  "type": "sriov",
  "cniVersion": "0.3.1",
  "name": "sriovnet-fxp",
  "ipam": {
    "type": "host-local",
    "subnet": "1.10.1.0/24",
    "routes": [{
      "dst": "0.0.0.0/0"
    }],
    "gateway": "1.10.1.1"
  }
}'
```
Multiple NADs such as the above can be created. 

## create sriov container 

Refer the NAD created above as an annotation. 
```
root@k8s-master:~/vsrx_sriov_fix# more test_pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: testpod1
  annotations:
    k8s.v1.cni.cncf.io/networks: sriovnet-fxp
spec:
  containers:
  - name: appcntr1
    image: centos/tools
    imagePullPolicy: IfNotPresent
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 300000; done;" ]
    resources:
      requests:
        intel.com/intel_sriov_netdevice: '1'
      limits:
        intel.com/intel_sriov_netdevice: '1'
```

### Verify the pod
```
kubectl exec -it <pod> sh
```
Verify if a new interface from the defined NAD has been attached. you can run `ifconfig`

Then ping the gateway 

```
64 bytes from 1.10.1.1: icmp_seq=8 ttl=64 time=5.78 ms
64 bytes from 1.10.1.1: icmp_seq=9 ttl=64 time=6.85 ms
64 bytes from 1.10.1.1: icmp_seq=10 ttl=64 time=9.77 ms
```

## Create a VM using Kubevirt
In this case considering vSRX to bootup using SRIOV DPDK interfaces. 

### Create NAD

Notice the sriov_dpdk used in the annotations. without this, kubevirt will throw an error when spinning up the VM. 
The error has been discussed in the earlier sections. 

Ensure to add respective vlan tags to the VF.

```
root@k8s-master:~/vsrx_sriov_fix# more sriov_nad_fxp.yml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriovnet-fxp
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_dpdk
spec:
  config: '{
  "type": "sriov",
  "cniVersion": "0.3.1",
  "name": "sriovnet-fxp",
  "vlan": 1000,
  "ipam": {
    "type": "host-local",
    "subnet": "1.10.1.0/24",
    "routes": [{
      "dst": "0.0.0.0/0"
    }],
    "gateway": "1.10.1.1"
  }
}'
```
similarly multiple NADs can be created.

### Create VM manifest file
 
In this case, using vSRX and attaching DPDK SRIOV interface.  
 
```
root@k8s-master:~/vsrx_sriov_fix# more vsrx-kubevirt.yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vsrx-sriov
spec:
  running: True
  template:
    metadata:
      labels:
        kubevirt.io/vm: vsrx-sriov
    spec:
      domain:
        ioThreadsPolicy: auto
        cpu:
          sockets: 1
          cores: 4
          threads: 1
        resources:
          requests:
            memory: 8Gi
            cpu: "4"
          limits:
            cpu: "4"
            memory: 8Gi
        devices:
          useVirtioTransitional: true
          disks:
            - name: boot
              disk:
                bus: virtio
          interfaces:
            - name: fxp
              sriov: {}
            - name: left
              sriov: {}
            - name: right
              sriov: {}
      volumes:
        - name: boot
          dataVolume:
            name: vsrx-sriov
      networks:
        - name: fxp
          multus:
            networkName: sriovnet-fxp
        - name: left
          multus:
            networkName: sriovnet-left
        - name: right
          multus:
            networkName: sriovnet-right
```

### Verify the VM

```
kubectl apply -f vsrx-kubevirt.yaml

root@k8s-master:~/vsrx_sriov_fix#  kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vsrx-sriov-6vp2z   1/1     Running   0          136m

root@k8s-master:~/vsrx_sriov_fix# kubectl get vm
NAME         AGE    STATUS    READY
vsrx-sriov   136m   Running   True
```

#### Login to the VM and verify NIC 
```
root@k8s-master:~/vsrx_sriov_fix# virtctl console vsrx-sriov
Successfully connected to vsrx-sriov console. The escape sequence is ^]

root> show chassis fpc pic-status
Slot 0   Online       FPC
  PIC 0  Online       VSRX DPDK GE

root> exit

root@:~ # vty fwdd


BSD platform (KVM virtual processor, 0MB memory, 16384KB flash)

FLOWD_VSRX( vty)#

FLOWD_VSRX( vty)# show nic config port 0
, 0
********************* Infos for port 0  *********************
MAC address: 02:09:C0:D2:B6:DF
Driver name: net_ixgbe_vf
Link status: up
Link speed: 10000 Mbps
Link duplex: full-duplex
Promiscuous mode: disabled
Allmulticast mode: enabled
VLAN offload: 3
Hash key size in bytes: 40
  ipv4
  ipv4-tcp
  ipv4-udp
  ipv6
  ipv6-tcp
  ipv6-udp
  unknown
  unknown
  unknown
Max possible RX queues: 4
Number of configured RX queues: 1
Max possible number of RXDs per queue: 4096
Min possible number of RXDs per queue: 32
RXDs number alignment: 8
Max possible TX queues: 4
Number of configured TX queues: 1
Max possible number of TXDs per queue: 4096
Min possible number of TXDs per queue: 32
TXDs number alignment: 8
```

## Backup content

### Install accelerated bridge CNI
```
git clone https://github.com/k8snetworkplumbingwg/accelerated-bridge-cni.git
kubectl apply -f images/k8s-v1.16/accelerated-bridge-cni-daemonset.yaml
```

### Validate
```
root@k8s-master:~/sriov_cni/accelerated-bridge-cni# kubectl get pods -A | grep accelerated-bridge
kube-system   kube-accelerated-bridge-cni-ds-amd64-q9grf   1/1     Running   0             35s
kube-system   kube-accelerated-bridge-cni-ds-amd64-tzmq2   1/1     Running   0             35s
kube-system   kube-accelerated-bridge-cni-ds-amd64-zf2zd   1/1     Running   0             35s
root@k8s-master:~/sriov_cni/accelerated-bridge-cni# kubectl get pods -A | grep sriov
kube-system   kube-sriov-cni-ds-amd64-gsjdq                1/1     Running   0             45m
kube-system   kube-sriov-cni-ds-amd64-hsw74                1/1     Running   0             45m
kube-system   kube-sriov-cni-ds-amd64-qzl99                1/1     Running   0             45m
```

configs for accelerated bridge. This needed for creating the Network attachment definitions etc. The SRIOV device plugin for discovery and report of VFs allocatable and accelerated bridge for VM NAD attachment. 
 
```
https://github.com/k8snetworkplumbingwg/accelerated-bridge-cni/blob/master/docs/configuration-reference.md
```

