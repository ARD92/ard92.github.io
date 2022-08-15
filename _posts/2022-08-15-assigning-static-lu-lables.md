---
layout: post
title: Assigning static LU labels
tags: junos mx crpd
---

There are scenarios where one may want to advertise a static LU label. In order to do that, a policy config is required on the DUT (MX/cRPD) . This policy must then be exported on the BGP session in order to take effect. 
The receiving router would receive a route <prefix, NH, label> binding (from peer or RR).
The policy config is part of the `from` stanza and the label allocation is is on `per-prefix` basis. Without this knob configured, the label allocation policy would be dynamic based on how LU works.

Consider the below scenario 
![topology](/images/static_lu_label.png)

- Here 21.21.21.21/32 is advertised from obce1 towards OB1
- The routes are propagated from OB1 towards RR and then further towards OB3
- At OB3, a policy is applied to allocate a static label 
- Receiver OB4 sees this route with static label 

## Configuration 

### Exact label 
```
root@ob3# show protocols bgp | display set
set protocols bgp group OB2 type internal
set protocols bgp group OB2 local-address 3.3.3.3
set protocols bgp group OB2 family inet unicast
set protocols bgp group OB2 family inet-vpn unicast
set protocols bgp group OB2 neighbor 2.2.2.2
set protocols bgp group OB4 type external
set protocols bgp group OB4 family inet labeled-unicast per-prefix-label
set protocols bgp group OB4 family inet-vpn unicast
set protocols bgp group OB4 export STATIC_LABEL
set protocols bgp group OB4 peer-as 55500
set protocols bgp group OB4 neighbor 192.168.3.2 local-address 192.168.3.1
set protocols bgp traceoptions file bgp.log
set protocols bgp traceoptions file size 100m
set protocols bgp traceoptions flag all
```
Notice the `per-prefix-label`. Without this labels would not be allocated. 

```
root@ob3# show policy-options policy-statement STATIC_LABEL | display set
set policy-options policy-statement STATIC_LABEL term 10 from route-filter 21.21.21.21/32 exact label 1000000
set policy-options policy-statement STATIC_LABEL term 10 then accept
```
By default, label block for static is from the range of 1M

### Range of labels
```
root@ob3# show route-filter-list STATIC_LABEL_RANGE | display set
set policy-options route-filter-list STATIC_LABEL_RANGE 31.31.31.0/30 orlonger label range 1000005:1000012

root@ob3# show policy-options policy-statement STATIC_LABEL term 20 | display set explicit
set policy-options policy-statement STATIC_LABEL term 20 from route-filter-list STATIC_LABEL_RANGE
set policy-options policy-statement STATIC_LABEL term 20 then accept
```

### Label range
```
root@ob3# run show mpls label usage
Label space Total   Available        Applications
LSI         999984  999976 (100.00%) BGP/LDP VPLS with no-tunnel-services, BGP L3VPN with vrf-table-label
Block       999984  999976 (100.00%) BGP/LDP VPLS with tunnel-services, BGP L2VPN
Dynamic     999984  999976 (100.00%) RSVP, LDP, PW, L3VPN, RSVP-P2MP, LDP-P2MP, MVPN, EVPN, BGP
Static      48576   48575  (100.00%) Static LSP, Static PW
Effective Ranges
Range name  Shared with Start   End
Dynamic     16      999999
Static      1000000 1048575
Configured Ranges
Range name  Shared with Start   End
Dynamic     16      999999
Static      1000000 1048575
```

In order to configure a different Static label block

```
root@ob4# set protocols mpls label-range static-label-range 800000 900000

[edit]
root@ob4# show | compare
[edit protocols mpls]
+    label-range {
+        static-label-range 800000 900000;
+    }
```

#### View the change
```
root@ob4# run show mpls label usage
Label space Total   Available        Applications
< ------ Snipped ----- >
Static      800000  900000
```
## Label allocation policy 

- Either an exact label per prefix or a range of labels for a subnet can be allocated.
- For example , if we need to allocate labels for a /30, then we need to reserve 7 labels
    - /30 (2^2) + /31 (2^1) + /32 (2^0) = 4+2+1 = 7 

## Outputs
```
root@ob2> show route 21.21.21.21

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

21.21.21.21/32     *[BGP/170] 00:57:07, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.1.1 via ob2_ob1

root@ob2> show route receive-protocol bgp 1.1.1.1 extensive
bgp.l3vpn.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)

* 100:100:10.10.10.10/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 100:100
     VPN Label: 23
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: I
     Communities: target:100:100

root@ob3# run show route 21.21.21.21

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

21.21.21.21/32     *[BGP/170] 00:58:25, localpref 100, from 2.2.2.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.2.2 via ob3_ob2, Push 23

root@ob3# run show route receive-protocol bgp 2.2.2.2 extensive

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
* 21.21.21.21/32 (1 entry, 1 announced)
     Accepted
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  2.2.2.2
     Originator ID: 1.1.1.1


root@ob3# run show route advertising-protocol bgp 192.168.3.2 extensive

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
* 21.21.21.21/32 (1 entry, 1 announced)
 BGP group OB4 type External
     Route Label: 1000000
     Nexthop: Self
     Flags: Nexthop Change
     AS path: [65500] I
     Communities: 100:100

root@ob4> show route receive-protocol bgp 192.168.3.1 extensive

inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
* 21.21.21.21/32 (1 entry, 1 announced)
     Accepted
     Route Label: 1000000
     Nexthop: 192.168.3.1
     AS path: 65500 I
     Communities: 100:100

```
