---
layout: post
title: Port mirroring and packet truncation 
tags: mx juniper junos
---
Juniper MX supports a variety of ways to perform port mirroring and the mirroring can be used in various use cases. 
- Mirror packets based on instance 
- Mirror packets based on firewall filter
- Mirror to multiple destination using next hop groups 

Additionally when performing mirroring (using the 1st 2 options), one can also set the max-packet-length in order to truncate the packet. Truncation of packets could be useful in scenarios where 1:1 sampling is performed and one would want to reduce the bandwidth consumed towards the analyzer. For example truncate the payload to reduce packet size.

## Configuring Port mirroring
Port mirror the packets to a destination port using firewall filters. When configuring firewall filters, refer to the below port mirroring instance  

```
root@vmx2# show forwarding-options
port-mirroring {
    instance {
        PM-1 {
            input {
                rate 1; << 1:1 sampling
                maximum-packet-length 65;
            }
            family inet {
                output {
                    interface gr-0/0/5.0 { << GRE encap
                        next-hop 200.1.1.1;
                    }
			  interface ge-0/0/5.0 { << non GRE ecap
		     		next-hop 200.2.1.1;
			  }
                }
            }
        }
    }
}

Bind the PM instance to FPC 

root@vmx2# show chassis fpc 0
pic 0 {
    tunnel-services {
        bandwidth 1g;
    }
}
port-mirror-instance PM1;
port-mirror-instance PM2;
```
Refer to the above created PM instance and bind the interface

```
set firewall family inet filter GTP-U-TRUNCATE term 10 then port-mirror-instance PM1
set firewall family inet filter GTP-U-TRUNCATE term 10 then discard

set interfaces ge-0/0/5 unit 50 family inet filter output GTP-U-TRUNCATE
```

## Mirror to multiple destinations using Next Hop groups 
By doing next hop groups we are looking up the forwarding next hop and directly spraying the packet. So for mirroring to multiple destinations this is more useful and efficient compared to filtering method.
In case the mirrored packets needs to be GRE encapsulated, then in next hop groups a `gr-` interface can be used instead of a `ge-` interface. This would encapsulate the packet with a GRE header. 

```
set firewall family inet filter GTP-U-NOTRUNCATE term 10 from destination-address 1.1.1.1/32 set firewall family inet filter GTP-U-NOTRUNCATE term 10 from port 2152
set firewall family inet filter GTP-U-NOTRUNCATE term 10 then count GTP-U-NOTRUNCATE_T10
set firewall family inet filter GTP-U-NOTRUNCATE term 10 then next-hop-group nhg-gtp-notruncate

set interfaces ge-0/0/1 passive-monitor-mode
set interfaces ge-0/0/1 unit 0 family inet filter input GTP-U-NOTRUNCATE
set interfaces ge-0/0/1 unit 0 family inet address 172.20.1.5/30
set interfaces ge-0/0/1 unit 0 family inet6

set forwarding-options next-hop-group nhg-gtp-notruncate group-type inet
set forwarding-options next-hop-group nhg-gtp-notruncate interface ge-0/0/5.50 next-hop 199.5.1.2
set forwarding-options next-hop-group nhg-gtp-notruncate interface ge-0/0/5.60 next-hop 199.5.2.2 
set forwarding-options next-hop-group nhg-gtp-notruncate interface ge-0/0/5.70 next-hop 199.5.3.2

set interfaces ge-0/0/5 flexible-vlan-tagging
set interfaces ge-0/0/5 unit 50 vlan-id 50
set interfaces ge-0/0/5 unit 50 family inet address 199.5.1.1/30
set interfaces ge-0/0/5 unit 60 vlan-id 60
set interfaces ge-0/0/5 unit 60 family inet address 199.5.2.1/30
set interfaces ge-0/0/5 unit 70 vlan-id 70
set interfaces ge-0/0/5 unit 70 family inet address 199.5.3.1/30
```

## What if we want to mirror inclduing l2 header  
When the destination is on local site, then the expectation is that collectors are the immediate next hop. i.e. they are directly connected. The same l2-mirror knob config you were using would work for this as well.

### Configuration
```
==== config port mirror =====
set forwarding-options port-mirroring instance TEST-L2-MIRROR input rate 1
set forwarding-options port-mirroring instance TEST-L2-MIRROR family any output interface ge-0/0/1.1
set chassis fpc 0 port-mirror-instance TEST-L2-MIRROR
 
==== output interface config ======
Note that I am leaving the interfaces untagged. So that I don’t modify the link. Native-vlan-id achieves this. If you don’t want multiple IFLs, then single unit 0 would work I believe.
 
set bridge-domains PM-INTF domain-type bridge
set bridge-domains PM-INTF vlan-id 1
set bridge-domains PM-INTF interface ge-0/0/1.1
 
set interfaces ge-0/0/1 flexible-vlan-tagging
set interfaces ge-0/0/1 native-vlan-id 1
set interfaces ge-0/0/1 encapsulation extended-vlan-bridge
set interfaces ge-0/0/1 unit 1 vlan-id 1
set interfaces ge-0/0/1 unit 1 family bridge
 
============= firewall filter ==========
set firewall family inet filter TEST-L2-MIRROR term 10 then count TEST-L2-MIRROR
set firewall family inet filter TEST-L2-MIRROR term 10 then port-mirror-instance TEST-L2-MIRROR
set firewall family inet filter TEST-L2-MIRROR term 10 then l2-mirror
 
 
======== add filter to input interface =======
set interfaces ge-0/0/0 passive-monitor-mode
set interfaces ge-0/0/0 unit 0 family inet filter input TEST-L2-MIRROR
set interfaces ge-0/0/0 unit 0 family inet address 10.1.1.1/30
``` 
 
 
### Captures 

```
=== start traffic with highlighted source/dest mac  ===
/app # ./go-packet-gen -S 2.2.2.2 -D 3.3.3.3 -t udp -m B6:44:66:5E:33:7C -M 02:aa:01:10:01:00 -i pkt1 -n 10 -d 443 -s 10-100
 
                      ____         ____            _        _      ____            __ _
                     / ___| ___   |  _ \ __ _  ___| | _____| |_   / ___|_ __ __ _ / _| |_ ___ _ __
                    | |  _ / _ \  | |_) / _ |/ __| |/ / _ \ __| | |   |  __/ _ | |_| __/ _ \  __|
                    | |_| | (_) | |  __/ (_| | (__|   <  __/ |_  | |___| | | (_| |  _| ||  __/ |
                     \____|\___/  |_|   \__,_|\___|_|\_\___|\__|  \____|_|  \__,_|_|  \__\___|_|
 
--> IP packet with udp
--> Sending packet..
 
===== tcpdump on collector ====
20:37:35.628091 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.55 > 3.3.3.3.443: UDP, length 9
20:37:35.841666 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.56 > 3.3.3.3.443: UDP, length 9
20:37:36.153754 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.57 > 3.3.3.3.443: UDP, length 9
20:37:36.402480 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.58 > 3.3.3.3.443: UDP, length 9
20:37:36.618045 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.59 > 3.3.3.3.443: UDP, length 9
20:37:36.836547 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.60 > 3.3.3.3.443: UDP, length 9
20:37:37.057531 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.61 > 3.3.3.3.443: UDP, length 9
20:37:37.277003 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.62 > 3.3.3.3.443: UDP, length 9
20:37:37.504175 b6:44:66:5e:33:7c > 02:aa:01:10:01:00, ethertype IPv4 (0x0800), length 51: 2.2.2.2.63 > 3.3.3.3.443: UDP, length 9
 
```

## Configuring Packet Truncation
By default when we choose the action `port-mirror` only packets with L3 header and above are mirrored. If packets with L2 header also needs to be mirrored then use action `then l2-mirror` . This will preserve the L2 header and then packet length can be set accordingly. 

Notice below that the maximum-packet-length is 65. This would mean 65B from the top would be passed and the rest would be truncated.
```
set firewall family inet filter GTP-U-TRUNCATE term 10 from destination-address 1.1.1.1/32
set firewall family inet filter GTP-U-TRUNCATE term 10 from port 2152
set firewall family inet filter GTP-U-TRUNCATE term 10 then count GTP-U-TRUNCATE_T10
set firewall family inet filter GTP-U-TRUNCATE term 10 then port-mirror-instance PM1
set firewall family inet filter GTP-U-TRUNCATE term 10 then discard

set forwarding-options port-mirroring instance PM1 input rate 1
set forwarding-options port-mirroring instance PM1 input maximum-packet-length 65
set forwarding-options port-mirroring instance PM1 family inet output interface ge-0/0/5.10 next-hop 199.1.1.2

set interfaces ge-0/0/5 flexible-vlan-tagging
set interfaces ge-0/0/5 unit 50 vlan-id 50
set interfaces ge-0/0/5 unit 50 family inet address 199.5.1.1/30
set interfaces ge-0/0/5 unit 50 family inet filter output GTP-U-TRUNCATE
```
![packettruncator](/images/packet_truncate_1.png)

### Validate packet truncation
#### Generate traffic 
Here I am generating traffic with payload `hellomynameisaravind`. The intention is to truncate the payload before sending it to analyzer.

```
go-packet-gen -S 100.2.1.1 -D 1.1.1.2 -t udp -s 2152 -d 2152 -m 12:40:CF:39:65:EF -M 02:aa:01:10:01:01 -i pktgen2 -p hellomynameisaravind -T 1000 -x 02aa011001011240cf3965ef08004500003000000000001194b764020101c0010102271001bb001c61cf 68656c6c6f6d796e616d65697361726176696e64 -n 10000
```
#### Check Ingress interface 
```
root@vmx2# run monitor interface ge-0/0/1
Interface: ge-0/0/1, Enabled, Link is Up
Encapsulation: Ethernet, Speed: 1000mbps
Traffic statistics:                                              Current delta
  Input bytes:                    335850 (6936 bps)                     [1764]
  Output bytes:                        0 (0 bps)                           [0]
  Input packets:                    3995 (10 pps)                         [21]
  Output packets:                      0 (0 pps)                           [0]
```

#### Check egress interface
```
root@vmx2# run monitor interface ge-0/0/5
Interface: ge-0/0/5, Enabled, Link is Up
Encapsulation: Ethernet, Speed: 1000mbps
Traffic statistics:                                              Current delta
  Input bytes:                      1560 (0 bps)                           [0]
  Output bytes:                   819996 (21016 bps)                    [5359]
  Input packets:                      24 (0 pps)                           [0]
  Output packets:                  10454 (33 pps)                         [69]
```

#### Check receiving analyzer port

##### VLAN 60 - No truncation 
```
/app # tcpdump -nei pm1.60
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on pm1.60, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:57:54.734871 02:aa:01:10:01:05 > c2:48:d9:26:14:29, ethertype IPv4 (0x0800), length 98: 100.2.1.1.2152 > 1.1.1.2.2152: UDP, length 56
12:57:54.854869 02:aa:01:10:01:05 > c2:48:d9:26:14:29, ethertype IPv4 (0x0800), length 98: 100.2.1.1.2152 > 1.1.1.2.2152: UDP, length 56
```
On VLAN 60, no truncation occurs. 
![packettruncator](/images/packet_truncate_2.png)

##### VLAN 10 - Truncate payload 
Notice that on VLAN 10 you would see `19 bytes missing`. Downstream nodes may detect this as a malformed packet and may drop it. It is of an expectation that the downsteam nodes can handle malformed packets.

```
/app # tcpdump -nei pm1.10
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode listening on pm1.10, link-type EN10MB (Ethernet), snapshot length 262144 bytes 12:58:25.527113 02:aa:01:10:01:05 > c2:48:d9:26:14:29, ethertype IPv4 (0x0800), length 79: truncated-ip - 19 bytes missing! 100.2.1.1.2152 > 1.1.1.2.2152: UDP, length 56
12:58:25.642835 02:aa:01:10:01:05 > c2:48:d9:26:14:29, e
```
![packettruncator](/images/packet_truncate_3.png)

## What if you want to do L2 mirroring ? i.e. interfaces are L2 interfaces both mirroring ports and mirror to ports 

```
====== pm output interface ====
root@map1# show interfaces ge-0/0/0 | display set
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation flexible-ethernet-services
set interfaces ge-0/0/0 unit 403 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 403 vlan-id 403
set interfaces ge-0/0/0 unit 403 family bridge filter output TEST-OUTPUT
 
root@map1# show interfaces ge-0/0/0 | display set
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation flexible-ethernet-services
set interfaces ge-0/0/0 unit 403 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 403 vlan-id 403
set interfaces ge-0/0/0 unit 403 family bridge filter output TEST-OUTPUT
 
 
====== input interface to mirror =====
root@map1# show interfaces ge-0/0/1 | display set
set interfaces ge-0/0/1 flexible-vlan-tagging
set interfaces ge-0/0/1 native-vlan-id 503
set interfaces ge-0/0/1 encapsulation flexible-ethernet-services
set interfaces ge-0/0/1 unit 503 encapsulation vlan-bridge
set interfaces ge-0/0/1 unit 503 vlan-id 503
set interfaces ge-0/0/1 unit 503 family bridge filter input PM
 
root@map1# show bridge-domains INPUT-PM | display set
set bridge-domains INPUT-PM vlan-id 503
set bridge-domains INPUT-PM interface ge-0/0/1.503
 
======== mirror config =========
 
root@map1# show forwarding-options | display set
set forwarding-options port-mirroring instance PM1 input rate 1
set forwarding-options port-mirroring instance PM1 family vpls output interface ge-0/0/0.403
set forwarding-options port-mirroring instance PM1 family vpls output no-filter-check
 
 
====== filter config ========
root@map1# show firewall family bridge filter PM | display set
set firewall family bridge filter PM term 10 then count count-pm-input
set firewall family bridge filter PM term 10 then port-mirror-instance PM1
set firewall family bridge filter PM term 10 then accept
 
[edit]
root@map1# show firewall family bridge filter TEST-OUTPUT | display set
set firewall family bridge filter TEST-OUTPUT term 10 then count pm-count-out
set firewall family bridge filter TEST-OUTPUT term 10 then accept
set firewall family bridge filter TEST-OUTPUT term 10 then log 
```

### Verification

Generate traffic
``` 
root@ubuntu:~/go-packet-crafter# ./go-packet-gen --smac 02:aa:01:40:01:00 --dmac 02:aa:01:10:02:01 -i eth1 -t udp -S 7.7.7.7 -D 18.18.18.18 -s 1023 -d 1088 -n 10
 
 
--> IP packet with udp
--> Sending packet..
 
root@map1# run show firewall
Filter: PM
Counters:
Name                                                                            Bytes              Packets
count-pm-input                                                                   2100                   35
 
Filter: TEST-OUTPUT
Counters:
Name                                                                            Bytes              Packets
pm-count-out                                                                      660                   11
 
tcpdump. On egress interface
 
16:01:16.007717 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.031690 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.055691 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.079606 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.103647 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.127704 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.152077 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.175748 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.199735 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
16:01:16.223768 02:aa:01:40:01:00 > 02:aa:01:10:02:01, ethertype IPv4 (0x0800), length 60: 7.7.7.7.1023 > 18.18.18.18.1088: UDP, length 9
```
## Troubleshooting commands

```
root@vmx2# run show forwarding-options port-mirroring
Instance Name: PM1
  Instance Id: 2
  Input parameters:
    Rate                  : 1
    Run-length            : 0
    Maximum-packet-length : 65
  Output parameters:
    Family              State     Destination          Next-hop
    inet                up        ge-0/0/5.10          199.1.1.2
 
Instance Name: PM2
  Instance Id: 3
  Input parameters:
    Rate                  : 1
    Run-length            : 0
    Maximum-packet-length : 0
  Output parameters:
    Family              State     Destination          Next-hop
    inet                up        ge-0/0/5.20          199.2.1.2
 

 
VMX-0(vmx2 vty)# show sample instance summary
Inst  Rate length Next-hop Clip-size Max-pps Class Proto  Ref-Inst Name Ref-inst-name ref-count
-----------------------------------------------------------------------------------------------
   1     1      1      784         0   65535     5     0      0 &global_instance           0
   1     1      1      785         0   65535     5     1      0 &global_instance           0
   1     1      1      786         0   65535     5     5      0 &global_instance           0
   1     1      1      787         0   65535     7     0      0 &global_instance           0
   1     1      1      788         0   65535     7     1      0 &global_instance           0
   1     1      1      789         0   65535     7     5      0 &global_instance           0
   2     1      0      781        65       0     2     0      0 PM1           0
   3     1      0      782         0       0     2     0      0 PM2           0
   4     1      0        0         0    1000     1     0      0 SAMPLE-1           0
 
 

VMX-0(vmx2 vty)# show filter nexthops
                           Name Protocol         Type Option Refcount  NH ID
------------------------------- -------- ------------ ------ --------  -----
             nhg-gtp-notruncate     IPv4 nexthop-group   0x00        3 16777286
                       nhg-rlt0     IPv4 nexthop-group   0x00        0    858



VMX-0(vmx2 vty)# show nhdb id 858 statistics
Nexthop Statistics:
Interface      NH ID Next Hop Addr    Output Pkts Pkt Rate    Output Bytes  Byte Rate   Protocol
------------ ------- --------------- ------------ -------- --------------- ---------- ---------- 
```

## References 
- [juniper-kb-article](https://supportportal.juniper.net/s/article/MX-Example-Configuring-port-mirroring-on-MX-devices?language=en_US)
- [juniper-documentation](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/port-mirroring-instances.html)

