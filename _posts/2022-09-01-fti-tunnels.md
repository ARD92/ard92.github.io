---
layout: post
title: Flexible tunnel interfaces
tags: MX linux crpd
---
## Topology

```
PE1 (vmx) ------------------ Host (crpd)
          0/0/0          fti1
```

## Configuration

Here VXLAN-GPE and VXLAN will interop. 

## vMX side 
vMX side, we will use vxlan-gpe with destination-udp-port `4789`. vxlan-gpe native port is `4790`. 

The FTI interfaces on Junos uses a psuedo MAC.

SOURCE_MAC = 00:00:5e:00:52:01
DEST_MAC = 00:00:5e:00:52:00

```
root@PE1# show interfaces lo0
unit 0 {
    family inet {
        address 1.1.1.1/32;
    }
}

root@PE1# show interfaces fti0
unit 3 {
    tunnel {
        encapsulation vxlan-gpe {
            source {
                address 1.1.1.1; << tunnel source address. Loopback
            }
            destination {
                address 1.10.10.10; << tunnel end point address
            }
            tunnel-endpoint vxlan; << type 
            destination-udp-port 4789; 
            vni 300;
        }
    }
    family inet {
        address 19.3.1.1/30;
    }
```

## cRPD/Linux host config

Due to the pseudo mac used on Junos side, we would need to configure the MAC accordingly on linux host side

SOURCE_MAC = 00:00:5e:00:52:00
DEST_MAC = 00:00:5e:00:52:01

A static arp needs to be added to the dest IP address with DEST_MAC

```
root@33a5c2250ee0:/# ip link add vxlan300 type vxlan id 300 dev fti1 dstport 4789 local 1.10.10.10 remote 1.1.1.1
root@33a5c2250ee0:/# ip link set up vxlan300
root@33a5c2250ee0:/# ip addr add 19.3.1.2/30 dev vxlan300
root@33a5c2250ee0:/# ip link set dev vxlan300 address  00:00:5e:00:52:00
```
Add an ARP entry, without this traffic would not reach and ping fails

```
root@33a5c2250ee0:/# arp -s 19.3.1.1 00:00:5e:00:52:01 -i vxlan300
```

## Verify
```
root@PE1# run ping 19.3.1.2
PING 19.3.1.2 (19.3.1.2): 56 data bytes
64 bytes from 19.3.1.2: icmp_seq=0 ttl=64 time=2.083 ms
64 bytes from 19.3.1.2: icmp_seq=1 ttl=64 time=0.784 ms
64 bytes from 19.3.1.2: icmp_seq=2 ttl=64 time=0.788 ms
64 bytes from 19.3.1.2: icmp_seq=3 ttl=64 time=0.836 ms
64 bytes from 19.3.1.2: icmp_seq=4 ttl=64 time=0.978 ms
64 bytes from 19.3.1.2: icmp_seq=5 ttl=64 time=0.622 ms
64 bytes from 19.3.1.2: icmp_seq=6 ttl=64 time=0.789 ms
64 bytes from 19.3.1.2: icmp_seq=7 ttl=64 time=0.859 ms
64 bytes from 19.3.1.2: icmp_seq=8 ttl=64 time=4.785 ms
64 bytes from 19.3.1.2: icmp_seq=9 ttl=64 time=1.168 ms

root@33a5c2250ee0:/# tcpdump -nei vxlan300
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vxlan300, link-type EN10MB (Ethernet), capture size 262144 bytes
01:56:35.601655 00:00:5e:00:52:01 > 00:00:5e:00:52:00, ethertype IPv4 (0x0800), length 98: 19.3.1.1 > 19.3.1.2: ICMP echo request, id 39192, seq 8, length 64
01:56:35.601705 00:00:5e:00:52:00 > 00:00:5e:00:52:01, ethertype IPv4 (0x0800), length 98: 19.3.1.2 > 19.3.1.1: ICMP echo reply, id 39192, seq 8, length 64
01:56:36.604763 00:00:5e:00:52:01 > 00:00:5e:00:52:00, ethertype IPv4 (0x0800), length 98: 19.3.1.1 > 19.3.1.2: ICMP echo request, id 39192, seq 9, length 64
```

## Package for programming vxlan
If the above is tried on cRPD, then a yang package is available [here](https://github.com/ARD92/crpd-unicast-vxlan) which can be used to program the vxlan tunnels.
However note that local address knob is missing which needs to be enhanced. 

## APIs
When this model is used to deploy, there are FTI tunnel APIs which are part of the pRPD package which can be used to deploy tunnels at scale. 
The pRPD IDLs can be downloaded from the Juniper downloads . Search as JET IDLs

## References
- [configuring-fti-interfaces](https://www.juniper.net/documentation/us/en/software/junos/interfaces-encryption/topics/topic-map/configuring-flexible-tunnel-interfaces.html)

