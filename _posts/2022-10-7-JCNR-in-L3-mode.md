---
layout: post
title: JCNR as a cell site router in O-RAN/V-RAN deployments in L3 model
tags: juniper linux kubernetes
---

## Topology considered 
Below topology is considered for the remainder of the post. 
![topology](/images/jcnr_l3_topo.png)

- JCNR in L3 mode
- multinode k8s cluster (master + 2 worker nodes)
- master node is not scheduled to run vrouter pods
    - master node runs cRPD in RR mode to exchange inet-unicast routes 
- each VM comes up with 3 interfaces
    - pod interface for internet connectivity
    - mgmt interface where all 3 VMs are part of the same subnet
    - core facing interface with same name and part of the same subnet across all 3 nodes. 

## Bring up testbed with VMs 
Follow [here](https://github.com/ARD92/kubevirt-manifests/tree/main/jcnr_l3_topology) to bring up 3 VMs to test JCNR vrouter for functional testing 

## Install kubernetes
Once the VMS are up. Install kubernetes. Follow the steps defined in this [link](https://ard92.github.io/2022/04/23/setting-up-k8s-in-ubuntu.html)
As an alternative you can install in one VM and copy the same disk to other PVs so that it can be faster. This process can be automated as part of CI/CD.

we need to ensure to install a primary CNI (flannel) and multus before proceeding to the next steps. 

## Iptable rules for forwarding within bridges

By default iptables will not forward packets if created as a bridge on kubevirt. Adding the below rule helps in forwarding. Similarly add `ACCEPT` to other bridges 
This will ensure packets between kubevirt VMs are passed.

```
iptables -A FORWARD -i k8s-l3br1 -o k8s-l3br1 -j ACCEPT
```

## VFs vs Virtio ? 
This setup uses virtio interfaces so you might need the module `uio_pci_generic`. In case it is missing, install using the below. You can also use SRIOV VFs accordingly.

### Load module
```
modprobe uio_pci_generic 
```
### If failed, install dependencies
```
apt install linux-modules-extra-5.4.0-128-generic
```
#### Validate
```
root@worker1:~# lsmod | grep uio_pci_generic
uio_pci_generic        16384  0
uio                    20480  1 uio_pci_generic
```

## Packages to install JCNR in L3 mode
Download the packages needed for JCNR. consists of 2 main things 
1. jcnr-vrouter images and helm charts
2. jcnr-cni images and helm charts 

we need to install both to ensure JCNR is working as expected.  

### Below steps are followed in order 
```
tar -xvf jcnr-R21.4.tar
tar xzvf jcnr-vrouter-21.4-181.tgz

# copy images to all worker nodes 
# Run the below to load images on all worker nodes 
sudo ctr -n k8s.io i import jcnr-vrouter-images.tar

# if docker is used, load images on worker nodes as below 
docker load -i jcnr-vrouter-images.tar.gz
```

## Modify helm chart values of jcnr-vrouter
1. since images are loaded, you can change image pull policy to `IfNotPresent`. Without this images would be constantly tried to pull from register and imagepullError might show up 
2. ensure you use the correct IP (along with subnet) and interfaces 

### Couple of gotchas! in the 21.4 release
1. if vrouter needs to come up on multiple nodes, then ensure all the interfaces are on same subnet.  
2. Interface name need not be same but GW from all nodes need to be same. This needs helm chart development 
3. only one core facing interface supported today. either a regular interface or a bond interface. Can have only a single IP

### Install vrouter 
```
cd jcnr-vrouter

helm install jcnr-vrouter .
```

#### Verify 
```
root@master:~/odu-pods# kubectl get pods -n contrail
NAME                                     READY   STATUS    RESTARTS   AGE
contrail-k8s-deployer-6c4444c79b-hgx5s   1/1     Running   0          12h
contrail-vrouter-nodes-42hnp             2/2     Running   0          12h
contrail-vrouter-nodes-655vk             2/2     Running   0          12h
contrail-vrouter-nodes-g72v9             2/2     Running   0          12h
```

### Install CRPD-CNI 
```
cd jcnr-cni 

helm install jcnr-cni .
```
#### Verify
```
root@master:~/odu-pods# kubectl get pods -n kube-system | grep crpd
kube-crpd-worker-ds-c9fs2        1/1     Running   1 (127m ago)   128m
kube-crpd-worker-ds-ddhgr        1/1     Running   1 (127m ago)   128m
kube-crpd-worker-ds-sjhbz        1/1     Running   1 (127m ago)   128m
```

### Verify if JCNR is up and running as expected 

#### configs 
Log into master and worker cRPD and verify configs 
```
kubectl exec -it <crpd-pod-id> -n kube-system cli 

root@master> show configuration | display set
set version 20210915.070446_builder.r1211817
set groups base apply-flags omit
set groups base apply-macro ht jcnr
<---- snipped ---->
```

#### Interfaces 
verify if vhost0 is created on cRPD and vif on vrouter 

##### cRPD
```
root@master> show interfaces routing
Interface        State Addresses
vhost0           Up    MPLS  enabled
                       ISO   enabled
                       INET  192.180.1.2
                       INET6 fe80::4026:c9ff:fefa:bb54
                       INET6 fe80::8c6:f8ff:fe98:aa21
```
##### vRouter 
Log into vrouter agent pod and verify interfaces 
```
kubectl exec -it <vrouter-agent-pod-id > -n contrail bash

root@worker1:/# vif --list
Vrouter Interface Table

Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
       Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
       D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
       Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
       Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
       Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
       HbsR=HBS Right Intf, Ig=Igmp Trap Enabled, Ml=MAC-IP Learning Enabled, Me=Multicast Enabled

vif0/0      PCI: 0000:03:00.0 (Speed 10000, Duplex 1) NH: 4
            Type:Physical HWaddr:5a:b6:18:73:91:05 IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:L3L2VpVofErMe QOS:-1 Ref:14
            RX device packets:106252  bytes:9379609 errors:0
            RX port   packets:106245 errors:0
            RX queue  packets:89628 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:03:00.0  Status: UP  Driver: net_virtio
            RX packets:106245  bytes:9378927 errors:0
            TX packets:2219244764  bytes:142034194813 errors:0
            Drops:0
            TX queue  packets:2219188982 errors:0
            TX port   packets:2218261786 errors:982978
            TX device packets:2218261793  bytes:141971284577 errors:0

vif0/1      PMD: vhost0 NH: 5
            Type:Host HWaddr:5a:b6:18:73:91:05 IPaddr:192.180.1.3
            IP6addr:fe80::58b6:18ff:fe73:9105
            Vrf:0 Mcast Vrf:65535 Flags:L3DEr QOS:-1 Ref:13
            RX device packets:94159  bytes:8459383 errors:0
            RX queue  packets:94159 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:94159  bytes:8459383 errors:0
            TX packets:114159  bytes:9711315 errors:0
            Drops:4
            TX queue  packets:114159 errors:0
            TX device packets:114159  bytes:9711315 errors:0

vif0/2      Socket: unix
            Type:Agent HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3Er QOS:-1 Ref:3
            RX port   packets:16469 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:16469  bytes:1701406 errors:0
            TX packets:12710  bytes:1370848 errors:0
            Drops:0

vif0/3      PMD: vhostblue-net-2ee68cf3-5be9-469 NH: 18
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:1.1.51.2
            Vrf:2 Mcast Vrf:2 Flags:PL3DEr QOS:-1 Ref:12
            RX port   packets:2238865888 errors:0
            RX queue  packets:2238864512 errors:1376
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 1376
            RX packets:2238864512  bytes:134331870720 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:19723834
```

#### Protocols 

##### ISIS and L-ISIS
This is verified from cRPD on worker nodes 

```
root@worker1> show isis adjacency
Interface             System         L State        Hold (secs) SNPA
vhost0                master         2  Up                   26  42:26:c9:fa:bb:54
vhost0                worker2        2  Up                   22  e:8c:cb:57:26:12

root@worker1> show isis route
 IS-IS routing table             Current version: L1: 0 L2: 65
IPv4/IPv6 Routes
----------------
Prefix             L Version   Metric Type Interface       NH   Via                 Backup Score
100.1.1.2/32       2      65       10 int  vhost0          IPV4 master
100.1.1.4/32       2      65       10 int  vhost0          IPV4 worker2
abcd::2/128        2      65       10 int  vhost0          IPV6 master
abcd::4/128        2      65       10 int  vhost0          IPV6 worker2

MPLS Routes
-----------
Label              L Version   Metric Type Interface       NH     Via
16/52              2      65        0 int  vhost0          MPLS   Direct->master(100.1.1.2)
16/56              2      65        0 int  vhost0          MPLS   Direct->master(100.1.1.2)
17/52              2      65        0 int  vhost0          MPLS   Direct->master(100.1.1.2)
17/56              2      65        0 int  vhost0          MPLS   Direct->master(100.1.1.2)
20/52              2      65        0 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
20/56              2      65        0 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
21/52              2      65        0 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
21/56              2      65        0 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
403002/52          2      65       10 int  vhost0          MPLS   master(100.1.1.2)
403002/56          2      65       10 int  vhost0          MPLS   Direct->master(100.1.1.2)
403004/52          2      65       10 int  vhost0          MPLS   worker2(100.1.1.4)
403004/56          2      65       10 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
402002/52          2      65       10 int  vhost0          MPLS   master(100.1.1.2)
402002/56          2      65       10 int  vhost0          MPLS   Direct->master(100.1.1.2)
402004/52          2      65       10 int  vhost0          MPLS   worker2(100.1.1.4)
402004/56          2      65       10 int  vhost0          MPLS   Direct->worker2(100.1.1.4)
IPv4/IPv6->MPLS Routes
----------------------
Prefix             L Version   Metric Type Interface       NH   Via                 Backup Score
100.1.1.2/32       2      65       10 int  vhost0          MPLS master(100.1.1.2)
100.1.1.4/32       2      65       10 int  vhost0          MPLS worker2(100.1.1.4)
abcd::2/128        2      65       10 int  vhost0          MPLS master(100.1.1.2)
abcd::4/128        2      65       10 int  vhost0          MPLS worker2(100.1.1.4)
```

##### BGP
BGP peering with master which runs RR to exchange routes.

```
root@worker1> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 1
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0
                       1          1          0          0          0          0
bgp.l3vpn-inet6.0
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
100.1.1.2             64512       1365       1368       0       1    10:11:16 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/0/0/0
  blue.inet.0: 1/1/1/0
```

##### Route tables
```
# If not inet.3 routes, then BGP L3VPN routes would be hidden
root@worker1> show route table inet.3

inet.3: 2 destinations, 4 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

100.1.1.2/32       *[LDP/9] 10:09:51, metric 1
                    >  to 192.180.1.2 via vhost0
                    [L-ISIS/14] 10:10:35, metric 10
                    >  to 192.180.1.2 via vhost0
100.1.1.4/32       *[LDP/9] 10:09:35, metric 1
                    >  to 192.180.1.4 via vhost0
                    [L-ISIS/14] 10:09:46, metric 10
                    >  to 192.180.1.4 via vhost0


root@worker1> show route table blue.inet.0

blue.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.51.0/30        *[Direct/0] 08:34:14
                    >  via vhostblue-net-2ee68cf3-5be9-469
1.1.51.2/32        *[Local/0] 08:34:14
                       Local via vhostblue-net-2ee68cf3-5be9-469
1.1.61.0/30        *[BGP/170] 08:34:10, localpref 100, from 100.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.180.1.4 via vhost0, Push 24
```
 
## Dpdk pod 

Create a pod which uses JCNR to forward packets with vrouter dataplane

### Create NAD
```
root@master:~/odu-pods# more nad.yml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: blue-net
spec:
  config: '{
    "cniVersion":"0.4.0",
    "name": "blue-net",
    "type": "jcnr",
    "args": {
      "vrfName": "blue",
      "vrfTarget": "11:11"
    },
    "kubeConfig":"/etc/kubernetes/kubelet.conf"
  }'
```

### Create dpdk pod1 
```
root@master:~/odu-pods# more pktgen_worker1.yml
apiVersion: v1
kind: Pod
metadata:
  name:   odu-l3-1
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "blue-net",
          "interface":"blue-net",
          "cni-args": {
            "mac": "aa:bb:cc:dd:ee:51",
            "dataplane": "vrouter",
            "ipConfig": {
              "ipv4": {
                "address":"1.1.51.2/30",
                "gateway":"1.1.51.1",
                "routes": [
                  "1.1.0.0/16"
                ]
              }
            }
          }
        }
      ]
spec:
  nodeName: worker1
  containers:
    - name: odu-l3
      image: svl-artifactory.juniper.net/junos-docker-local/warthog/pktgen19116:
20210303
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

### Create dpdk pod on worker2

```
root@master:~/odu-pods# more pktgen_worker2.yml
apiVersion: v1
kind: Pod
metadata:
  name:   odu-l3-2
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "blue-net",
          "interface":"blue-net",
          "cni-args": {
            "mac": "aa:bb:cc:dd:ee:61",
            "dataplane": "vrouter",
            "ipConfig": {
              "ipv4": {
                "address":"1.1.61.2/30",
                "gateway":"1.1.61.1",
                "routes": [
                  "1.1.0.0/16"
                ]
              }
            }
          }
        }
      ]
spec:
  containers:
    - name: odu-l3-2
      image: svl-artifactory.juniper.net/junos-docker-local/warthog/pktgen19116:
20210303
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

### Start Traffic

#### Log into dpdk workload 
##### Pod1 (worker1)
```
kubectl exec -it <dpdk-pod-1> bash

# look at dpdk pod interfaces 
{"vhost-adaptor-path":"/dpdk/vhost-blue-net.sock","vhost-adaptor-mode":"client",
"ipv4-address":"1.1.51.2/30","mac-address":"aa:bb:cc:dd:ee:51"}

# edit the pktgen file 
set 0 src mac aa:bb:cc:dd:ee:51
set 0 src ip 1.1.51.2/30
set 0 dst mac aa:bb:cc:dd:ee:61
set 0 dst ip 1.1.61.2/30
set 0 burst 32
set 0 rate 1
set 0 size 64

# start packet gen on dpdk-pod-1 
rm -rf /dpdk/vhost-blue-net.sock
/usr/local/bin/pktgen  -l 4-5 -n 4 --socket-mem 1024 --no-pci --file-prefix=pktgen --vdev=net_virtio_user1,mac=aa:bb:cc:dd:ee:51,path=/dpdk/vhost-blue-net.sock,server=1 --single-file-segments -- -P -m 5.0 -f pktgen.pkt
```

##### Pod2 (worker2)
```
kubectl exec -it <dpdk-pod-2> bash

# look at dpdk pod interfacs 
{"vhost-adaptor-path":"/dpdk/vhost-blue-net.sock","vhost-adaptor-mode":"client",
"ipv4-address":"1.1.61.2/30","mac-address":"aa:bb:cc:dd:ee:61"}

# edit the pktgen file. Notice the dmac. this is psuedo mac and not the actual mac of the receiver
set 0 src mac aa:bb:cc:dd:ee:61
set 0 src ip 1.1.61.2/30
set 0 dst mac 00:00:5e:00:01:00
set 0 dst ip 1.1.51.2/30
set 0 burst 32
set 0 rate 1
set 0 size 64

# start the packetgen app
rm -rf /dpdk/vhost-blue-net.sock
/usr/local/bin/pktgen  -l 4-5 -n 4 --socket-mem 1024 --no-pci --file-prefix=pktgen --vdev=net_virtio_user1,mac=aa:bb:cc:dd:ee:61,path=/dpdk/vhost-blue-net.sock,server=1 --single-file-segments -- -P -m 5.0 -f pktgen.pkt
```
##### Generate traffic 
from both packetgen app type "start 0" to start the traffic and "stop 0" to stop traffic.

TX 
```
/ Ports 0-0 of 1   <Main Page>  Copyright (c) <2010-2020>, Intel Corporation
  Flags:Port        : P------Single      :0
Link State          :         <UP-10000-FD>      ---Total Rate---
Pkts/s Max/Rx       :                   0/0                   0/0
       Max/Tx       :         149664/149664         149664/149664
MBits/s Rx/Tx       :                 0/100                 0/100
Broadcast           :                     0
Multicast           :                     0
Sizes 64            :                     0
      65-127        :                     0
      128-255       :                     0
      256-511       :                     0
      512-1023      :                     0
      1024-1518     :                     0
Runts/Jumbos        :                   0/0
ARP/ICMP Pkts       :                   0/0
Errors Rx/Tx        :                   0/0
Total Rx Pkts       :                     0
      Tx Pkts       :                996544
      Rx MBs        :                     0
      Tx MBs        :                   669
                    :
Pattern Type        :               abcd...
Tx Count/% Rate     :           Forever /1%
Pkt Size/Tx Burst   :             64 /   32
TTL/Port Src/Dest   :        64/ 1234/ 5678
Pkt Type:VLAN ID    :       IPv4 / TCP:0001
802.1p CoS/DSCP/IPP :             0/  0/  0
VxLAN Flg/Grp/vid   :      0000/    0/    0
IP  Destination     :              1.1.61.2
    Source          :           1.1.51.2/30
MAC Destination     :     00:00:5e:00:01:00
    Source          :     aa:bb:cc:dd:ee:51
PCI Vendor/Addr     :     0000:0000/00:00.0
```

RX
```
\ Ports 0-0 of 1   <Main Page>  Copyright (c) <2010-2020>, Intel Corporation
  Flags:Port        : P------Single      :0
Link State          :         <UP-10000-FD>      ---Total Rate---
Pkts/s Max/Rx       :         150080/149568         150080/149568
       Max/Tx       :                   0/0                   0/0
MBits/s Rx/Tx       :                 100/0                 100/0
Broadcast           :                     0
Multicast           :                     0
Sizes 64            :               5633667
      65-127        :                     0
      128-255       :                     0
      256-511       :                     0
      512-1023      :                     0
      1024-1518     :                     0
Runts/Jumbos        :                   0/0
ARP/ICMP Pkts       :                   0/0
Errors Rx/Tx        :                   0/0
Total Rx Pkts       :               5499683
      Tx Pkts       :                     0
      Rx MBs        :                  3695
      Tx MBs        :                     0
                    :
Pattern Type        :               abcd...
Tx Count/% Rate     :           Forever /1%
Pkt Size/Tx Burst   :             64 /   32
TTL/Port Src/Dest   :        64/ 1234/ 5678
Pkt Type:VLAN ID    :       IPv4 / TCP:0001
802.1p CoS/DSCP/IPP :             0/  0/  0
VxLAN Flg/Grp/vid   :      0000/    0/    0
IP  Destination     :              1.1.51.2
    Source          :           1.1.61.2/30
MAC Destination     :     00:00:5e:00:01:00
    Source          :     aa:bb:cc:dd:ee:61
PCI Vendor/Addr     :     0000:0000/00:00.0
```

##### Verify tcpdump captures on underlying host bridge 
```
root@k8s-worker2:~# tcpdump -nei k8s-l3brvrouter

05:51:49.166689 5a:b6:18:73:91:05 > 0e:8c:cb:57:26:12, ethertype MPLS unicast (0x8847), length 64: MPLS (label 24, exp 0, [S], ttl 63) 1.1.51.2.1234 > 1.1.61.2.5678: Flags [.], seq 0:6, ack 1, win 8192, length 6
05:51:49.166690 5a:b6:18:73:91:05 > 0e:8c:cb:57:26:12, ethertype MPLS unicast (0x8847), length 64: MPLS (label 24, exp 0, [S], ttl 63) 1.1.51.2.1234 > 1.1.61.2.5678: Flags [.], seq 0:6, ack 1, win 8192, length 6
05:51:49.166692 5a:b6:18:73:91:05 > 0e:8c:cb:57:26:12, ethertype MPLS unicast (0x8847), length 64: MPLS (label 24, exp 0, [S], ttl 63) 1.1.51.2.1234 > 1.1.61.2.5678: Flags [.], seq 0:6, ack 1, win 8192, length 6
05:51:49.166693 5a:b6:18:73:91:05 > 0e:8c:cb:57:26:12, ethertype MPLS unicast (0x8847), length 64: MPLS (label 24, exp 0, [S], ttl 63) 1.1.51.2.1234 > 1.1.61.2.5678: Flags [.], seq 0:6, ack 1, win 8192, length 6
05:51:49.166695 5a:b6:18:73:91:05 > 0e:8c:cb:57:26:12, ethertype MPLS unicast (0x8847), length 64: MPLS (label 24, exp 0, [S], ttl 63) 1.1.51.2.1234 > 1.1.61.2.5678: Flags [.], seq 0:6, ack 1, win 8192, length 6
```

## Troubleshooting

1. If Pod doesn't start since not scheduled . You can check `kubectl describe pods <pod name> -n contrail

    ```
    Events:
      Type     Reason            Age   From               Message
      ----     ------            ----  ----               -------
      Warning  FailedScheduling  51s   default-scheduler  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
    ``` 

    Edit the templates/jcnr-vrouter.yaml and remove the node scheduler which is pointed to master node.

2. level=error msg="invalid ip enp3s0 for interface enp3s0.

    Ensure the value of `vhost_interface_ipv4` contains IP along with subnet. Example 
    ```
    vhost_interface_ipv4: "192.170.2.3/28"
    ```

3. cRPD failing due to missing label 
    The helm charts has several tolerations and nodeSelectors used
    
    Label the nodes
    ```
    root@master:~# kubectl label nodes master node-role.kubernetes.io/master=
    node/master labeled
    root@master:~# kubectl label nodes worker1 node-role.kubernetes.io/worker=
    node/worker1 labeled
    root@master:~# kubectl label nodes worker2 node-role.kubernetes.io/worker=
    ```

4. cRPD errors due to license failure

    a. If license failures occur, then cRPD would not start at all. This can be identified by looking at docker logs for cRPD 
        - find crpd running on node 
            ```
            kubectl get pods -A -o wide | grep crpd
            ```
    b. login to the node
    c. find the crpd container id
        ```
        docker ps | grep crpd
        ```
    d. check logs 
        if docker
        ```
        docker logs <crpd-container-id>
        ```
    
        if crictl is used 
        ```
        crictl logs <crpd-container-id>
        ```
    e. add new license by converting to base64
        ```
        # store license in a file named license.txt
        
        #Run the below to generate a base64 value   
        base64 license.txt

        #save the generated value into jcnr-secrets.yaml file
        ```
5. cRPD started but no configs present
    This is because jdeployer failed provisioning the configs due to errors present in templates. This could be mostly in the file `jcnr-secrets` because thats the user fed file. 
    the remaining templates files are pre-built so they are usually syntax checked and correct. Verify if there are additional characters or intendation issues. These would cause the 
    configs to be rejected by mgd. 
    
    the logs would be pointed at `/var/log/jcnr-cni.log`

    The cRPD configs would be stored in `/etc/crpd`. 

    try deleting and reinstalling the helm chart to see if it has fixed the issue 
    ```
    # on worker/master where cRPD is deployed
    rm -rf /etc/crpd

    # from master
    helm uninstall jcnr-cni 

    # from master
    helm install jcnr-cni .
    ```
6. Specifiying incorrect cni args when creating pods
    if incorrect arguments or errors noticed in pod yamls when creating workloads to use JCNR, the pods may not start. we have taken dpdk packetgenpod as example.
    The errors would report that cni failed to add interface due to extra characters or incorrect/unrecognizable arguments. correcting the workload manifest should start the pods

7. dpdk packetgen app traffic not passing
    check the smac and dmac address. If this is incorrect traffic would not pass and would get dropped. 
    in L3 mode ensure the dmac used is `00:00:5e:00:01:00`. without this no traffic would be passed. 

8. BGP sessions dont come up with worker crpd and master crpd RR.
    - enable traceoptions for bgp on the crpd to identify the issue. 
    -  sometimes the TCP port could cause an issue. Try deleting the TCP port for BGP connection
    ```
    deactivate groups base protocols bgp group CNI tcp-connect-port
    ```

