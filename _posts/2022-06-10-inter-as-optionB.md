---
layout: post
title: Inter AS Option B 
tags: juniper junos crpd mx
---

There are multiple ways one can connect to 2 ASBRs. Option A, B, C ,AB.. In this post we will talk about configuring Option B style. 

- Option A:  Back to Back VRFs and each ASBR thinks the other is a CE
- Option B: MP-eBGP between ASBRs. No VRF exists on ASBR , they only exchange labeled routes
- Option C: MP-eBGP between RRs 

## Why Option B ? 
- Scalable compared to option A. only one link between ASBR and carries all service routes on top of it. 
- labels are allocated but no VRF exists. Only label swap states are installed. So all VRFs/service routes are carried over the single link 
- No need to exchange other SPs loopback address as in case of Option C when peering between RRs.

## Topology
Let us consider the below topology 

![topology](/images/optionb_ebgp.png)

Here OB1 and OB4 are the PE and OB2 and OB3 are the ASBRs. 

### Config on OB1
```
set interfaces lo unit 0 family inet address 1.1.1.1/32
set policy-options policy-statement ADD_CE2_COMM term 10 then community add ADD_CE2_COMM
set policy-options policy-statement ADD_CE2_COMM term 10 then accept
set policy-options policy-statement ADD_VPN1_COMM term 10 then community add ORG_COMM
set policy-options policy-statement ADD_VPN1_COMM term 10 then accept
set policy-options community ADD_CE2_COMM members origin:100.1.2.2:3
set policy-options community ORG_COMM members origin:100.1.1.2:3
set routing-instances VPN1 instance-type vrf
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 type external
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 import ADD_VPN1_COMM
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 family inet unicast
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 peer-as 64001
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 neighbor 100.1.1.2 local-address 100.1.1.1
set routing-instances VPN1 protocols bgp group PE-CE-CE2 type external
set routing-instances VPN1 protocols bgp group PE-CE-CE2 import ADD_CE2_COMM
set routing-instances VPN1 protocols bgp group PE-CE-CE2 family inet unicast
set routing-instances VPN1 protocols bgp group PE-CE-CE2 peer-as 64002
set routing-instances VPN1 protocols bgp group PE-CE-CE2 neighbor 100.1.2.2 local-address 100.1.2.1
set routing-instances VPN1 interface ob1_obce11
set routing-instances VPN1 interface ob1_obce12
set routing-instances VPN1 route-distinguisher 100:100
set routing-instances VPN1 vrf-target target:100:100
set routing-instances VPN1 vrf-table-label
set routing-instances VPN2 instance-type vrf
set routing-instances VPN2 route-distinguisher 200:100
set routing-instances VPN2 vrf-target target:200:100
set routing-instances VPN2 vrf-table-label
set routing-instances VPN3 instance-type vrf
set routing-instances VPN3 interface ob1_obce13
set routing-instances VPN3 route-distinguisher 300:100
set routing-instances VPN3 vrf-target target:300:100
set routing-instances VPN3 vrf-table-label
set routing-options router-id 1.1.1.1
set routing-options autonomous-system 65001
set protocols bgp group OB2 type internal
set protocols bgp group OB2 traceoptions file bgp.log
set protocols bgp group OB2 traceoptions file size 10m
set protocols bgp group OB2 traceoptions flag all
set protocols bgp group OB2 family inet-vpn unicast per-prefix-label
set protocols bgp group OB2 family route-target
set protocols bgp group OB2 neighbor 2.2.2.2 local-address 1.1.1.1
set protocols ldp interface ob1_ob2
set protocols mpls interface all
set protocols ospf area 0.0.0.100 interface ob1_ob2
set protocols ospf area 0.0.0.100 interface lo.0 passive
```

### Config on OB4
```
set interfaces lo unit 0 family inet address 4.4.4.4/32
set interfaces lo unit 0 family inet
set interfaces lo unit 0
set routing-instances VPN1 instance-type vrf
set routing-instances VPN1 interface ob4_obce41
set routing-instances VPN1 route-distinguisher 100:101
set routing-instances VPN1 vrf-target target:100:100
set routing-instances VPN1 vrf-table-label
set routing-instances VPN2 instance-type vrf
set routing-instances VPN2 interface ob4_obce42
set routing-instances VPN2 route-distinguisher 200:101
set routing-instances VPN2 vrf-target target:200:100
set routing-instances VPN2 vrf-table-label
set routing-instances VPN3 instance-type vrf
set routing-instances VPN3 interface ob4_obce43
set routing-instances VPN3 route-distinguisher 300:101
set routing-instances VPN3 vrf-target target:500:100
set routing-instances VPN3 vrf-table-label
set routing-options router-id 4.4.4.4
set routing-options autonomous-system 65002
set protocols bgp group OB3 type internal
set protocols bgp group OB3 family inet-vpn unicast
set protocols bgp group OB3 neighbor 3.3.3.3 local-address 4.4.4.4
set protocols bgp group OB3 neighbor 3.3.3.3
set protocols ldp interface ob4_ob3
set protocols mpls interface all
set protocols ospf area 0.0.0.200 interface ob4_ob3
set protocols ospf area 0.0.0.200 interface lo.0 passive
set protocols ospf area 0.0.0.200 interface lo.0
```

### Config on OB2

This is the ASBR config. Notice that no `routing-instances` would be configured. 
For every incoming label from PE there would be an outgoing label towards the far end ASBR. Since in most cases, label per VRF is allocated, the `swap` label would remain one per VRF as well.

```
set interfaces lo unit 0 family inet address 2.2.2.2/32
set routing-options router-id 2.2.2.2
set routing-options autonomous-system 65001
set protocols bgp group AS65002 type external
set protocols bgp group AS65002 local-address 192.168.2.2
set protocols bgp group AS65002 family inet-vpn unicast
set protocols bgp group AS65002 peer-as 65002
set protocols bgp group AS65002 local-as 65001
set protocols bgp group AS65002 neighbor 192.168.2.1
set protocols bgp group OB1 type internal
set protocols bgp group OB1 export NH_SELF
set protocols bgp group OB1 cluster 1.1.1.1
set protocols bgp group OB1 neighbor 1.1.1.1 local-address 2.2.2.2
set protocols ldp interface ob2_ob1
set protocols mpls interface all
set protocols ospf area 0.0.0.100 interface ob2_ob1
set protocols ospf area 0.0.0.100 interface lo.0 passive
``` 

### Config on OB3
```
root@ob3> show configuration | display set
set version "20220501.164742__cd-builder.r1255973 [_cd-builder]"
set interfaces lo unit 0 family inet address 3.3.3.3/32
set routing-options router-id 3.3.3.3
set routing-options autonomous-system 65002
set protocols bgp group AS65001 type external
set protocols bgp group AS65001 description "option b ebgp"
set protocols bgp group AS65001 local-address 192.168.2.1
set protocols bgp group AS65001 family inet-vpn unicast
set protocols bgp group AS65001 peer-as 65001
set protocols bgp group AS65001 local-as 65002
set protocols bgp group AS65001 neighbor 192.168.2.2
set protocols bgp group OB4 type internal
set protocols bgp group OB4 family inet-vpn unicast
set protocols bgp group OB4 export NH_SELF
set protocols bgp group OB4 neighbor 4.4.4.4 local-address 3.3.3.3
set protocols ldp interface ob3_ob4
set protocols mpls interface all
set protocols ospf area 0.0.0.200 interface ob3_ob4
set protocols ospf area 0.0.0.200 interface lo.0 passive
```

## Validation 

### Bgp summary
```
root@ob2> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0
                       3          3          0          0          0          0
bgp.l3vpn.0
                      12         12          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.1               65001       9116       9150       0       0 2d 20:47:48 Establ
  bgp.l3vpn.0: 9/9/9/0
192.168.2.1           65002         14         13       0       0        3:46 Establ
  bgp.l3vpn.0: 3/3/3/0
```

### BGP.L3VPN.0 table
```
[edit]
root@ob2# run show route table bgp.l3vpn.0

bgp.l3vpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

300:101:100.4.3.0/30
                   *[BGP/170] 00:04:23, localpref 100
                      AS path: 65002 I, validation-state: unverified
                    >  to 192.168.2.1 via ob2_ob3, Push 154
100:100:100.1.1.0/30
                   *[BGP/170] 2d 20:48:40, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 36
100:100:100.1.2.0/30
                   *[BGP/170] 2d 20:48:40, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 41
100:100:100.11.0.1/32
                   *[BGP/170] 2d 20:48:40, localpref 100, from 1.1.1.1
                      AS path: 64001 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 36
100:100:100.11.0.2/32
                   *[BGP/170] 2d 20:48:40, localpref 100, from 1.1.1.1
                      AS path: 64001 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 36
100:100:100.11.0.3/32
                   *[BGP/170] 2d 20:48:40, localpref 100, from 1.1.1.1
                      AS path: 64001 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 36
100:100:100.12.0.1/32
                   *[BGP/170] 2d 20:48:39, localpref 100, from 1.1.1.1
                      AS path: 64002 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 39
100:100:100.12.0.2/32
                   *[BGP/170] 2d 20:48:39, localpref 100, from 1.1.1.1
                      AS path: 64002 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 39
100:100:100.12.0.3/32
                   *[BGP/170] 2d 20:48:39, localpref 100, from 1.1.1.1
                      AS path: 64002 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 39
100:100:172.17.0.0/16
                   *[BGP/170] 2d 20:48:39, localpref 100, from 1.1.1.1
                      AS path: 64001 I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1, Push 36
100:101:100.4.1.0/30
                   *[BGP/170] 00:04:23, localpref 100
                      AS path: 65002 I, validation-state: unverified
                    >  to 192.168.2.1 via ob2_ob3, Push 152
200:101:100.4.2.0/30
                   *[BGP/170] 00:04:23, localpref 100
                      AS path: 65002 I, validation-state: unverified
                    >  to 192.168.2.1 via ob2_ob3, Push 153
```

### MPLS table

Notice the label swaps installed. All the VPN routes now appear only in `bgp.l3vpn.0` table and labels will be swapped and advertised to the far end ASBR.  

```
root@ob2# run show route table mpls.0

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 2d 20:49:27, metric 1
                       Receive
1                  *[MPLS/0] 2d 20:49:27, metric 1
                       Receive
2                  *[MPLS/0] 2d 20:49:27, metric 1
                       Receive
13                 *[MPLS/0] 2d 20:49:27, metric 1
                       Receive
16                 *[LDP/9] 2d 20:49:10, metric 1
                    >  to 192.168.1.1 via ob2_ob1, Pop
16(S=0)            *[LDP/9] 2d 20:49:10, metric 1
                    >  to 192.168.1.1 via ob2_ob1, Pop
24                 *[VPN/170] 00:05:09, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 36
25                 *[VPN/170] 00:05:09, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 41
26                 *[VPN/170] 00:05:09, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 39
27                 *[VPN/170] 00:04:53
                    >  to 192.168.2.1 via ob2_ob3, Swap 153
28                 *[VPN/170] 00:04:53
                    >  to 192.168.2.1 via ob2_ob3, Swap 152
```

Similar outputs are seen on the other end and traffic would pass E2E 

### Routes advertised to ASBR
```
root@ob2> show route advertising-protocol bgp 192.168.2.1 extensive

bgp.l3vpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
* 100:100:100.1.1.0/30 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 20
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] I
     Communities: target:100:100

* 100:100:100.1.2.0/30 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 22
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] I
     Communities: target:100:100

* 100:100:100.11.0.1/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 20
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.11.0.2/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 20
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.11.0.3/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 20
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.12.0.1/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 21
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64002 I
     Communities: target:100:100 origin:100.1.2.2:3
```

### Test E2E Traffic 

Ping CE-CE 
```
root@obce11:/# ping 100.4.1.1 -I 100.11.0.1
PING 100.4.1.1 (100.4.1.1) from 100.11.0.1 : 56(84) bytes of data.
64 bytes from 100.4.1.1: icmp_seq=1 ttl=61 time=0.085 ms
64 bytes from 100.4.1.1: icmp_seq=2 ttl=61 time=0.093 ms
64 bytes from 100.4.1.1: icmp_seq=3 ttl=61 time=0.103 ms
```
