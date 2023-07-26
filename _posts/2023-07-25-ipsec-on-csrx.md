---
layout: post
title: IPSEC on cSRX leveraging service chaining  
tags: kubernetes
---

In this post, let us see how to configure IPSEC on Juniper cSRX. Read the earlier post [here](https://ard92.github.io/2023/07/20/service-chaining-csrx-jcnr.html) on service chaining leveraging JCNR.  This post assumes basic topology is up and running following the mentioned link. This assumes kernel interfaces as it is meant only for functional testing. If need for performance, we should change to dpdk dataplane adn virtio-vhost interface or a dedicated dpdk vf/pf. 

## Topology considered

![topology](/images/jcnr_csrx_ipsec.png)

## Getting started

You can find the configs described below [here](https://github.com/ARD92/kubevirt-manifests/tree/main/jcnr_csrx_svc_chain) 


## NAD definition

### File nad-trust.yaml
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

### File nad-untrust.yaml
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

### File cSRX
```
root@master:~/23.2/csrx# more csrx-kernel-ipsec.yaml
# data plane can be linux, dpdk or veth
# interface type can be virtio or veth or dpdk
---
apiVersion: v1
kind: Pod
metadata:
  name: csrx-ipsec
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "net-trust",
          "interface":"eth1",
          "cni-args": {
            "mac":"aa:bb:cc:dd:11:12",
            "interfaceType":"veth",
            "rd": "11:11",
            "dataplane":"linux",
            "ipConfig":{
              "ipv4":{
                "address":"1.30.1.2/30",
                "gateway":"1.30.1.1"
              }
            }
          }
        },
        {
          "name": "net-untrust",
          "interface":"eth2",
          "cni-args": {
            "mac":"aa:bb:cc:dd:11:21",
            "interfaceType": "veth",
            "rd": "10:10",
            "dataplane":"linux",
            "ipConfig":{
              "ipv4":{
                "address":"1.31.1.2/30",
                "gateway":"1.31.1.1"
              }
            }
          }
        }
      ]
spec:
  containers:
  - name: csrx-ipsec
    securityContext:
       privileged: true
    image: csrx:23.3-20230721.0229_RELEASE_233_THROTTLE
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
    - name: CSRX_AUTO_ASSIGN_IP
      value: "yes"
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
        type: Directory
```

### Apply the files
```
kubectl apply -f nad-trust.yaml
kubectl apply -f nad-untrust.yaml
kubectl apply -f csrx-kernel-ipsec.yaml
```

### Add routes respectively 
Currently this is not orchestrated and shows a manual approach to solve the problem 

1. we need to redirect traffic to destination towards cSRX. So add a route with nexthop towards cSRX on trust side 
    ```
    set routing-instances trust routing-options static route 102.102.102.0/24 qualified-next-hop 1.32.1.2 interface jvketh1-72ba1c3
    ```
2. Similarly do it on reverse direction from untrust side
    ```
    set routing-instances untrust routing-options static route 101.101.101.0/24 qualified-next-hop 1.33.1.2 interface jvketh2-72ba1c3
    ```
3. Add IPSEC configs on cSRX 
    - traffic selectors are mandatory. Once this is configured traffic on one direction is automatically added 
    - reverse direction route needs to be added statically 
    - IPSEC cannot work in secure wire mode and hence must be static 

### IPSEC config on cSRX
```
set interfaces ge-0/0/0 unit 0 family inet address 1.30.1.2/30
set interfaces ge-0/0/1 unit 0 family inet address 1.31.1.2/30
set interfaces st0 unit 0 family inet

set security ike proposal ike-phase1-proposal authentication-method pre-shared-keys
set security ike proposal ike-phase1-proposal dh-group group5
set security ike proposal ike-phase1-proposal authentication-algorithm sha-256
set security ike proposal ike-phase1-proposal encryption-algorithm aes-256-cbc
set security ike proposal ike-phase1-proposal lifetime-seconds 3600
set security ike policy ike-phase1-policy proposals ike-phase1-proposal
set security ike policy ike-phase1-policy pre-shared-key ascii-text "$9$zt3l3AuIRhev8FnNVsYoaApu0RcSyev8XO1NVYoDj.P5F9AyrKv8X"
set security ike gateway remote ike-policy ike-phase1-policy
set security ike gateway remote address 171.1.1.1
set security ike gateway remote external-interface ge-0/0/1.0
set security ike gateway remote local-address 1.31.1.2
set security ike gateway remote version v2-only

set security ipsec proposal ipsec-phase2-proposal protocol esp
set security ipsec proposal ipsec-phase2-proposal authentication-algorithm hmac-sha-256-128
set security ipsec proposal ipsec-phase2-proposal encryption-algorithm aes-256-cbc
set security ipsec proposal ipsec-phase2-proposal lifetime-seconds 18000
set security ipsec policy ipsec-phase2-policy perfect-forward-secrecy keys group5
set security ipsec policy ipsec-phase2-policy proposals ipsec-phase2-proposal
set security ipsec vpn ipsec-remote bind-interface st0.0
set security ipsec vpn ipsec-remote ike gateway remote
set security ipsec vpn ipsec-remote ike ipsec-policy ipsec-phase2-policy
set security ipsec vpn ipsec-remote traffic-selector ts1 local-ip 180.1.1.0/24
set security ipsec vpn ipsec-remote traffic-selector ts1 remote-ip 172.1.1.0/24
set security ipsec vpn ipsec-remote establish-tunnels immediately

set security policies default-policy permit-all

set security zones security-zone untrust host-inbound-traffic system-services all
set security zones security-zone untrust host-inbound-traffic protocols all
set security zones security-zone untrust interfaces ge-0/0/1.0
set security zones security-zone untrust interfaces st0.0
set security zones security-zone trust host-inbound-traffic system-services all
set security zones security-zone trust host-inbound-traffic protocols all
set security zones security-zone trust interfaces ge-0/0/0.0
```

Once the above are done below would be the state. 

### Verification 

#### E2E Ping
```
root@ubuntu:/# ping 102.102.102.1 -I 101.101.101.1 <<< E2E ping.
PING 102.102.102.1 (102.102.102.1) from 101.101.101.1 : 56(84) bytes of data.
64 bytes from 102.102.102.1: icmp_seq=1 ttl=61 time=13.6 ms
64 bytes from 102.102.102.1: icmp_seq=2 ttl=61 time=2.27 ms
```

#### Route table information
```
# on JCNR
root@master> show route table trust.inet.0

trust.inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.30.1.0/30        *[Direct/0] 03:36:05
                    >  via jvketh1-72ba1c3
1.30.1.2/32        *[Local/0] 03:36:05
                       Local via jvketh1-72ba1c3
1.32.1.2/32        *[Static/5] 03:22:03
                    >  via jvketh1-72ba1c3
101.101.101.1/32   *[BGP/170] 03:36:03, localpref 100
                      AS path: 65003 I, validation-state: unverified
                    >  to 180.1.1.1 via ens4
102.102.102.0/24   *[Static/5] 03:19:27
                    >  via jvketh1-72ba1c3
180.1.1.0/30       *[Direct/0] 03:36:05
                    >  via ens4
180.1.1.2/32       *[Local/0] 03:36:05
                       Local via ens4


root@master> show route table untrust.inet.0

untrust.inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.31.1.0/30        *[Direct/0] 03:35:41
                    >  via jvketh2-72ba1c3
1.31.1.2/32        *[Local/0] 03:35:41
                       Local via jvketh2-72ba1c3
1.33.1.2/32        *[Static/5] 03:20:36
                    >  via jvketh2-72ba1c3
101.101.101.0/24   *[Static/5] 03:17:07
                    >  via jvketh2-72ba1c3
171.1.1.0/24       *[BGP/170] 03:35:35, localpref 100
                      AS path: 65002 I, validation-state: unverified
                    >  to 181.1.1.1 via ens5
181.1.1.0/30       *[Direct/0] 03:35:41
                    >  via ens5
181.1.1.2/32       *[Local/0] 03:35:41
                       Local via ens5

# On cSRX
root@csrx-ipsec# run show route forwarding-table
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
0.0.0.0            perm     0                    dscd     2517     1
1.30.1.0/30        perm     0                    rslv     2004     1 ge-0/0/0.0
1.30.1.1           perm     0 1.30.1.1          ucast     5501     1 ge-0/0/0.0
1.30.1.2           perm     0 1.30.1.2           locl     2001     1
1.30.1.3           perm     0                    bcst     2002     1 ge-0/0/0.0
1.31.1.0/30        perm     0                    rslv     2009     1 ge-0/0/1.0
1.31.1.1           perm     0 1.31.1.1          ucast     5500     1 ge-0/0/1.0
1.31.1.2           perm     0 1.31.1.2           locl     2006     1
1.31.1.3           perm     0                    bcst     2007     1 ge-0/0/1.0
1.32.1.0/30        perm     0                    rslv     2015     1 ge-0/0/0.0
1.32.1.2           perm     0 1.32.1.2           locl     2012     1
1.32.1.3           perm     0                    bcst     2013     1 ge-0/0/0.0
1.33.1.0/30        perm     0                    rslv     2020     1 ge-0/0/1.0
1.33.1.2           perm     0 1.33.1.2           locl     2017     1
1.33.1.3           perm     0                    bcst     2018     1 ge-0/0/1.0
101.101.101/24     perm     0 1.30.1.1          ucast     5501     1 ge-0/0/0.0
102.102.102/24     perm     0                   ucast     2011     1 st0.0
171.1.1.2          perm     0 1.31.1.1          ucast     5500     1 ge-0/0/1.0
```

#### Ipsec related 
```
root@csrx-ipsec# run show security ike security-associations
Index   State  Initiator cookie  Responder cookie  Mode           Remote Address
39      UP     5795cb15a66efded  3df78315d5322f8b  IKEv2          171.1.1.2

[edit]
root@csrx-ipsec# run show security ipsec security-associations
  Total active tunnels: 1     Total IPsec sas: 1
  ID      Algorithm       SPI      Life:sec/kb  Mon lsys Port  Gateway
  <500003 ESP:aes-cbc-256/sha256 0x93be8146 4743/ unlim - root 500 171.1.1.2
  >500003 ESP:aes-cbc-256/sha256 0x6c2491f7 4743/ unlim - root 500 171.1.1.2

root@csrx-ipsec# run show security ipsec statistics
ESP Statistics:
  Encrypted bytes:         14689428
  Decrypted bytes:          7909692
  Encrypted packets:          94163
  Decrypted packets:          94163
AH Statistics:
  Input bytes:                    0
  Output bytes:                   0
  Input packets:                  0
  Output packets:                 0
Errors:
  AH authentication failures: 0, Replay errors: 0
  ESP authentication failures: 0, ESP decryption failures: 0
  Bad headers: 0, Bad trailers: 0
  Invalid SPI: 0, TS check fail: 129
  Exceeds tunnel MTU: 0
  Discarded: 129
```
