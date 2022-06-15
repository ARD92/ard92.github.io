---
layout: post
title: Aggregate labels to improve label scale on OptionB routers
tags: crpd junos juniper mx
---

Aggregate labels are used in scenarios where the PE allocates label per prefix instead of label per VRF. In Junos the standard practice is to use `vrf-table-label` which allocates label per VRF. But there could be a different vendor device or a legacy device which could do per prefix label. In such scenarios, when the ASBR sees the VPN routes, it will continue to allocate labels for every prefix and the label table will reach max capacity. In order to efficiently handle that and optimize, there is  a knob called `aggregate label` which can be used based on origin community. The origin community needs to be added on the PE before advertising and can be done at per CE level within the VRF. Once the ASBR receives the origin community, it will allocate the same label for all the routes arriving with the same origin community ensuring the label scale is intact.

Label allocation per CE is important because of lookups. If we allocate origin community per VRF, as long as the router does IP lookup, it should be good else packets may get dropped without knowing which interface to exit out of.

## Topology 
![topology](/images/aggregate_label.png)

## Generate per prefix label

Generate per prefix label for the PE 
```
root@ob1# show protocols bgp
group OB2 {
    type internal;
    traceoptions {
        file bgp.log size 10m;
        flag all;
    }
    family inet-vpn {
        unicast {
            per-prefix-label;
        }
    }
    inactive: family route-target;
    neighbor 2.2.2.2 {
        local-address 1.1.1.1;
    }
}
```

## Create VPNs
```
set routing-instances VPN1 instance-type vrf
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 type external
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 family inet unicast
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 peer-as 64001
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 neighbor 100.1.1.2 local-address 100.1.1.1
set routing-instances VPN1 protocols bgp group PE-CE-CE2 type external
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
```

## Verify labels advertised 

Verify if per prefix labels are advertised to ASBR 
```
root@ob1# run show route advertising-protocol bgp 2.2.2.2 extensive

VPN1.inet.0: 12 destinations, 15 routes (12 active, 0 holddown, 0 hidden)
* 100.1.1.0/30 (2 entries, 1 announced)
 BGP group OB2 type Internal
     Route Distinguisher: 100:100
     VPN Label: 51
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65001] I
     Communities: target:100:100

* 100.1.2.0/30 (2 entries, 1 announced)
 BGP group OB2 type Internal
     Route Distinguisher: 100:100
     VPN Label: 56
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65001] I
     Communities: target:100:100

* 100.11.0.1/32 (1 entry, 1 announced)
 BGP group OB2 type Internal
     Route Distinguisher: 100:100
     VPN Label: 51
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100.11.0.2/32 (1 entry, 1 announced)
 BGP group OB2 type Internal
     Route Distinguisher: 100:100
     VPN Label: 52
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3
```

## Create Origin communities 
```
set policy-options policy-statement ADD_CE2_COMM term 10 then community add ADD_CE2_COMM
set policy-options policy-statement ADD_CE2_COMM term 10 then accept
set policy-options policy-statement ADD_VPN1_COMM term 10 then community add ORG_COMM
set policy-options policy-statement ADD_VPN1_COMM term 10 then accept
set policy-options community ADD_CE2_COMM members origin:100.1.2.2:3
set policy-options community ORG_COMM members origin:100.1.1.2:3
```

## Create VPNs

Add Origin communities for per CE on the VRF
notice import policies `ADD_VPN1_COMM` and `ADD_CE2_COMM`

```
set routing-instances VPN1 protocols bgp group PE-CE-VPN1 import ADD_VPN1_COMM
set routing-instances VPN1 protocols bgp group PE-CE-CE2 import ADD_CE2_COMM
```

## On ASBR

Configure the aggregate label on the receiving session from PE

```
set protocols bgp group OB1 type internal
set protocols bgp group OB1 family inet-vpn unicast aggregate-label community ORG_COMM
set protocols bgp group OB1 export NH_SELF
set protocols bgp group OB1 cluster 1.1.1.1
set protocols bgp group OB1 neighbor 1.1.1.1 local-address 2.2.2.2

set policy-options community ORG_COMM members origin:100.1.1.2:3
set policy-options community ORG_COMM members origin:100.1.2.2:3
```

### Verify the labels

Note the labels being the same for all the prefixes received from per CE 

```
root@ob2> show route receive-protocol bgp 1.1.1.1 extensive

inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)

inet.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)

bgp.l3vpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
* 100:100:100.1.1.0/30 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 51
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: I
     Communities: target:100:100

* 100:100:100.1.2.0/30 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 56
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: I
     Communities: target:100:100

* 100:100:100.11.0.1/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 51
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.11.0.2/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 52
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.11.0.3/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 53
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64001 I
     Communities: target:100:100 origin:100.1.1.2:3

* 100:100:100.12.0.1/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 54
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64002 I
     Communities: target:100:100 origin:100.1.2.2:3

* 100:100:100.12.0.2/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 55
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64002 I
     Communities: target:100:100 origin:100.1.2.2:3

* 100:100:100.12.0.3/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 56
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64002 I
     Communities: target:100:100 origin:100.1.2.2:3

* 100:100:172.17.0.0/16 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 57
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: 64001 I
     Communities: target:100:100 origin:100.1.1.2:3
``` 

### MPLS route tables
```
root@ob2> show route table mpls.0

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 2d 03:06:09, metric 1
                       Receive
1                  *[MPLS/0] 2d 03:06:09, metric 1
                       Receive
2                  *[MPLS/0] 2d 03:06:09, metric 1
                       Receive
13                 *[MPLS/0] 2d 03:06:09, metric 1
                       Receive
16                 *[VPN/170] 2d 03:05:53
                    >  to 192.168.2.1 via ob2_ob3, Swap 161
17                 *[VPN/170] 2d 03:05:53
                    >  to 192.168.2.1 via ob2_ob3, Swap 162
18                 *[VPN/170] 2d 03:05:53
                    >  to 192.168.2.1 via ob2_ob3, Swap 163
19                 *[LDP/9] 2d 03:05:50, metric 1
                    >  to 192.168.1.1 via ob2_ob1, Pop
19(S=0)            *[LDP/9] 2d 03:05:50, metric 1
                    >  to 192.168.1.1 via ob2_ob1, Pop
20                 *[VPN/170] 2d 02:53:43, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 51
21                 *[VPN/170] 2d 02:53:43, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 54
22                 *[VPN/170] 2d 02:53:43, metric2 1, from 1.1.1.1
                    >  to 192.168.1.1 via ob2_ob1, Swap 56
```

### Routes advertised to far end
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

* 100:100:100.12.0.2/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 21
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64002 I
     Communities: target:100:100 origin:100.1.2.2:3

* 100:100:100.12.0.3/32 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 21
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64002 I
     Communities: target:100:100 origin:100.1.2.2:3

* 100:100:172.17.0.0/16 (1 entry, 1 announced)
 BGP group AS65002 type External
     Route Distinguisher: 100:100
     VPN Label: 20
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65001] 64001 I
     Communities: target:100:100 origin:100.1.1.2:3
```

