---
layout: post
title: Service chaining JCNR and cSRX to offer security services 
tags: kubernetes
---

## Containerized SRX and Integration with Juniper Cloud Native Router 

### Topology
![topology](/images/jcnr_csrx.png)

### Prerequisites 
* Ensure vfio-pci drivers are installed and NICS can be bound to the vfio-pci drivers so that JCNR can leverage it 
    This installs dpdk pktgen on the vm/bms which may not be needed
    ```
    git clone https://github.com/ARD92/dpdk_pktgen.git
    ./install-dpdk-pktgen.sh 
    ```
* Install Kubernetes 
    ```
    git clone https://github.com/ARD92/kubevirt-manifests.git
    cd install-k8s
    chmod +x install-k8s.sh
    ./install-k8s.sh
    ```
* Install Multus
    ```
    git clone https://github.com/k8snetworkplumbingwg/multus-cni.git && cd multus-cni
    cat ./deployments/multus-daemonset-thick.yml | kubectl apply -f -
    ``` 
* Install JCNR 
    Follow official juniper documentation [here](https://www.juniper.net/documentation/us/en/software/cloud-native-router23.2/cloud-native-router-deployment-guide/topics/concept/cnr-install-jcnr.html)

Steps following assumes that the above has been completed and you have an all in one kubernetes cluster. Either on a bare metal or on a VM 

### Network attachment definition

### Save the below into a file nad-trust.yaml
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: net-trust
spec:
  config: '{
    "cniVersion":"0.4.0",
    "name": "net-trust",
    "type": "jcnr",
    "args": {
      "vrfName": "trust",
      "vrfTarget": "11:11"
    },
    "kubeConfig":"/etc/kubernetes/kubelet.conf"
  }'
```

### Save the below into a file nad-untrust.yaml
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: net-untrust
spec:
  config: '{
    "cniVersion":"0.4.0",
    "name": "net-untrust",
    "type": "jcnr",
    "args": {
      "vrfName": "untrust",
      "vrfTarget": "10:10"
    },
    "kubeConfig":"/etc/kubernetes/kubelet.conf"
  }'
```

### Apply the NAD 
```
kubectl apply -f nad-trust.yaml
kubectl apply -f nad-untrust.yaml
```

### cSRX Manifest file 
Below creates cSRX with kernel mode. Save the below to csrx.yaml 
```
---
apiVersion: v1
kind: Pod
metadata:
  name: csrx
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "net-trust",
          "interface":"eth1",
          "cni-args": {
            "mac":"aa:bb:cc:dd:01:12",
            "interfaceType":"veth",
            "rd": "11:11",
            "dataplane":"dpdk",
            "ipConfig":{
              "ipv4":{
                "address":"1.20.1.2/30",
                "gateway":"1.20.1.1"
              }
            }
          }
        },
        {
          "name": "net-untrust",
          "interface":"eth2",
          "cni-args": {
            "mac":"aa:bb:cc:dd:01:21",
            "interfaceType": "veth",
            "rd": "10:10",
            "dataplane":"dpdk",
            "ipConfig":{
              "ipv4":{
                "address":"1.21.1.2/30",
                "gateway":"1.21.1.1"
              }
            }
          }
        }
      ]
spec:
  containers:
  - name: csrx1
    securityContext:
       privileged: true
    image: csrx:23.2R1.13
    imagePullPolicy: IfNotPresent
    env:
    - name: CSRX_SIZE
      value: "large"
    - name: CSRX_HUGEPAGES
      value: "no"
    - name: CSRX_PACKET_DRIVER
      value: "interrupt"
    - name: CSRX_FORWARD_MODE
      value: "routing"
      #- name: CSRX_AUTO_ASSIGN_IP
      #value: "yes"
    - name: CSRX_MGMT_PORT_REORDER
      value: "no"
    - name: CSRX_TCP_CKSUM_CALC
      value: "yes"
    - name: CSRX_LICENSE_FILE
      value: "/var/jail/.csrx_license"
    - name: CSRX_JUNOS_CONFIG
      value: "/var/jail/csrx_config"
    - name: CSRX_CTRL_CPU
      value: "0x01"
    - name: CSRX_DATA_CPU
      value: "0x05"
    volumeMounts:
    - name: disk
      mountPath: "/dev"
  volumes:
  - name: disk
    hostPath:
      path: /dev
      type: Director
```

### Apply cSRX manifest
```
kubectl apply -f csrx.yaml
```

### Validate the interfaces 
```
root@csrx:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.81  netmask 255.255.255.0  broadcast 10.244.0.255
        ether 16:f5:a9:7f:0d:3a  txqueuelen 0  (Ethernet)
        RX packets 9  bytes 830 (830.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15  bytes 1358 (1.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether aa:bb:cc:dd:01:12  txqueuelen 0  (Ethernet)
        RX packets 171650  bytes 258235769 (258.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 83924  bytes 5563556 (5.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether aa:bb:cc:dd:01:21  txqueuelen 0  (Ethernet)
        RX packets 83961  bytes 5566390 (5.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 170651  bytes 258108443 (258.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
pfe_tun: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# tap0 will be bound to eth1 
tap0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 1.20.1.1  netmask 255.255.255.252  broadcast 1.20.1.3
        inet6 fe80::a8bb:ccff:fedd:112  prefixlen 64  scopeid 0x20<link>
        ether aa:bb:cc:dd:01:12  txqueuelen 1000  (Ethernet)
        RX packets 9  bytes 378 (378.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 24  bytes 1280 (1.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# tap1 will be bound to eth2 
tap1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 1.21.1.1  netmask 255.255.255.252  broadcast 1.21.1.3
        inet6 fe80::a8bb:ccff:fedd:121  prefixlen 64  scopeid 0x20<link>
        ether aa:bb:cc:dd:01:21  txqueuelen 1000  (Ethernet)
        RX packets 11  bytes 462 (462.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 28  bytes 1456 (1.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### Configure cSRX 
```
root@csrx# show | display set
set version 20230616.051920_builder.r1345402
set system root-authentication encrypted-password *disabled*
set system services ssh root-login allow
set interfaces ge-0/0/0 unit 0 family inet address 1.20.1.1/30
set interfaces ge-0/0/1 unit 0 family inet address 1.21.1.1/30
set routing-options static route 181.1.1.1/32 next-hop 1.21.1.2/32
set routing-options static route 180.1.1.1/32 next-hop 1.20.1.2/32
set security policies default-policy permit-all
set security zones security-zone untrust interfaces ge-0/0/1.0 host-inbound-traffic system-services all
set security zones security-zone untrust interfaces ge-0/0/1.0 host-inbound-traffic protocols all
set security zones security-zone trust interfaces ge-0/0/0.0 host-inbound-traffic protocols all
```

### Configure JCNR with correct routes 
```
set routing-instances trust routing-options static route 181.1.1.1/32 next-hop 1.20.1.1
set routing-instances untrust routing-options static route 180.1.1.1/32 next-hop 1.21.1.1
```

### Pass traffic E2E 
```
root@ubuntu:~# ping 181.1.1.1
PING 181.1.1.1 (181.1.1.1) 56(84) bytes of data.
64 bytes from 181.1.1.1: icmp_seq=1 ttl=61 time=8.53 ms
64 bytes from 181.1.1.1: icmp_seq=2 ttl=61 time=1.77 ms
64 bytes from 181.1.1.1: icmp_seq=3 ttl=61 time=1.60 ms
64 bytes from 181.1.1.1: icmp_seq=4 ttl=61 time=1.19 ms


root@csrx> show security flow session
Session ID: 1071, Policy name: default-policy-logical-system-00/2, Timeout: 4, Session State: Valid
  In: 180.1.1.1/52 --> 181.1.1.1/1;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84,
  Out: 181.1.1.1/1 --> 180.1.1.1/52;icmp, Conn Tag: 0x0, If: ge-0/0/1.0, Pkts: 1, Bytes: 84,

Session ID: 1072, Policy name: default-policy-logical-system-00/2, Timeout: 4, Session State: Valid
  In: 180.1.1.1/52 --> 181.1.1.1/2;icmp, Conn Tag: 0x0, If: ge-0/0/0.0, Pkts: 1, Bytes: 84,
  Out: 181.1.1.1/2 --> 180.1.1.1/52;icmp, Conn Tag: 0x0, If: ge-0/0/1.0, Pkts: 1, Bytes: 84,
Total sessions: 2
```

### Debugging
* Make sure interface names are in format eth1, eth2 ... . If we use something else, this will fail
    ```
    root@csrx:/# cat /var/local/csrxrc
    export CSRX_SIZE=large
    export CSRX_PACKET_DRIVER=interrupt
    export CSRX_CTRL_CPU=0x1
    export CSRX_DATA_CPU=0x2
    export HOST_CSRX_CTRL_CPU=0x01
    export HOST_CSRX_DATA_CPU=0x05
    export CSRX_FORWARD_MODE=routing
    export CSRX_SWG_MODE=no
    export CSRX_ARP_TIMEOUT=1200000
    export CSRX_NDP_TIMEOUT=1200000
    export CSRX_HUGEPAGES=no
    export CSRX_USER_DATA=no
    export CSRX_PORT_NUM=3
    export CSRX_AUTO_ASSIGN_IP=no
    export CSRX_LICENSE_FILE=/var/jail/.csrx_license
    export CSRX_MGMT_PORT_REORDER=no
    export CSRX_TCP_CKSUM_CALC=yes
    export CSRX_PKID_BINDKEY=/var/db/pbk
    
    root@csrx:/#  more /var/local/csrx_base_cfg.xml
    <?xml version="1.0" encoding="UTF-8"?>
    <csrx mode="L3" vdev-mode="2">
      <logical-cplane-core-mask>0x1</logical-cplane-core-mask>
      <logical-flt-core-mask>0x2</logical-flt-core-mask>
      <host-cplane-core-mask>0x01</host-cplane-core-mask>
      <host-flt-core-mask>0x05</host-flt-core-mask>
      <dpdk-socket-mem>500</dpdk-socket-mem>
      <jumbo-frames-support></jumbo-frames-support>
      <arp-timeout-in-ms>1200000</arp-timeout-in-ms>
      <disable-hugepages>1</disable-hugepages>
      <recompute-tcp-checksum>yes</recompute-tcp-checksum>
      <interfaces>
          <interface>
               <eth-dev>eth1</eth-dev>
           </interface>
          <interface>
               <eth-dev>eth2</eth-dev>
           </interface>
       </interfaces>
    </csrx>

    root@csrx:/# cat /var/local/csrx_data_interfaces
    eth1
    eth2
    ```
* If tap interfaces are not created 
    For every eth interface, there will be a tap interface. Make sure they come up, else cSRX flowd does not receive the packets 

* Make sure you do not re-order your ports. you can use the below env flags
    ```
    CSRX_MGMT_PORT_REORDER: "No"
    ```

* To login to cSRX PFE
    ```
    root@csrx:/# vty 127.0.0.1
    TOR platform (VMWare virtual processor, 0MB memory, 16384KB flash)
    FLOWD_CSRX_L(vty)#
    ```

* If traffic is arrving on eth interface but not being passed to flowd on tap interface, check for license
    ```
    root@csrx# run show system license
    License usage:
                                     Licensed     Licensed    Licensed
                                      Feature      Feature     Feature
      Feature name                       used    installed      needed    Expiry
      anti_spam_key_sbl                     0            1           0    2024-04-16 00:00:00 UTC
      idp-sig                               0            1           0    2024-04-16 00:00:00 UTC
      appid-sig                             0            1           0    2024-04-16 00:00:00 UTC
      av_key_sophos_engine                  0            1           0    2024-04-16 00:00:00 UTC
      wf_key_websense_ewf                   0            1           0    2024-04-16 00:00:00 UTC
      cSRX                                  1            1           0    2024-04-16 00:00:00 UTC
    
    Licenses installed:
      License identifier: E20210416001
      License version: 4
      Order Type: commercial
      Software Serial Number: BYOL04162021
      Customer ID: JUNIPER-CSRX
      Features:
        av_key_sophos_engine - Anti Virus with Sophos Engine
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
        wf_key_websense_ewf - Web Filtering EWF
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
        anti_spam_key_sbl - Anti-Spam
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
        appid-sig        - APPID Signature
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
        cSRX             - Containerized Firewall
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
        idp-sig          - IDP Signature
          date-based, 2021-04-16 00:00:00 UTC - 2024-04-16 00:00:00 UTC
    ```
*   ports bound to JCNR
    ```
    /var/lib/contrail/ports
    ```
