---
layout: post
title: MPLS anti spoofing
tags: junos mx
---

Typically in requirements of security during connecting across 2 administrative domains , Service providers prefer option A. Option B however solves the scaling issues and allowing the ASBRs not to have any VRF configs.
The one link carries all service routes on top of it. Read [here](https://ard92.github.io/2022/06/10/inter-as-optionB.html) for more information on Inter AS option B (RFC 4364). 
scenarios where incorrect  RD advertisements or spoofed labels occur, the networks are prone to attacks.  The MPLS anti spoofing feature allows the ASBR to validate the label and ensures the packet is received from the correct interface. 
An MFI instance would be created where the interface would reside. The packets would undergo redirection to the instance where the labels and interface over which the packet arrived would be validated. This instance would be of type `mpls-forwarding`


## Topology 
![topology](/images/mpls_antispoof.png)

## Configs
```
root@P1# show | compare
[edit]
+  routing-instances {
+      spoof-check {
+          instance-type mpls-forwarding;
+          interface ge-0/0/1.0; << untrusted interface
+      }
+  }


[edit protocols bgp group UNDERLAY-P2 neighbor 10.1.1.7]
+     forwarding-context spoof-check; << enable forwarding context to verify 


root@P1# show protocols mpls
interface ge-0/0/0.0; << note that the untrusted interface is not present in mpls.0 table. It will be part of the new instance “spoof-check” 
``` 

## Verify

### Mpls.0 table
```
root@P1> show route table mpls.0

mpls.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 2d 08:35:01, metric 1
                       to table inet.0
0(S=0)             *[MPLS/0] 2d 08:35:01, metric 1
                       to table mpls.0
1                  *[MPLS/0] 2d 08:35:01, metric 1
                       Receive
2                  *[MPLS/0] 2d 08:35:01, metric 1
                       to table inet6.0
2(S=0)             *[MPLS/0] 2d 08:35:01, metric 1
                       to table mpls.0
13                 *[MPLS/0] 2d 08:35:01, metric 1
                       Receive
299776             *[LDP/9] 2d 08:35:00, metric 1
                    >  to 10.1.1.0 via ge-0/0/0.0, Pop
299776(S=0)        *[LDP/9] 2d 08:35:00, metric 1
                    >  to 10.1.1.0 via ge-0/0/0.0, Pop
299824             *[VPN/170] 2d 06:46:53
                    >  to 10.1.1.7 via ge-0/0/1.0, Swap 299840
```

### Verify spoof check table 
```
root@P1> show route table spoof-check.inet.0

spoof-check.inet.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.1.1.6/31        *[Direct/0] 2d 06:50:49
                    >  via ge-0/0/1.0
10.1.1.6/32        *[Local/0] 2d 06:50:49
                       Local via ge-0/0/1.0

root@P1> show route table spoof-check.mpls.0

spoof-check.mpls.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

299840             *[VPN/170] 2d 06:10:21, metric2 1, from 1.1.1.1
                    >  to 10.1.1.0 via ge-0/0/0.0, Swap 16
1000000            *[VPN/170] 2d 06:14:47
                       receive table inet.0, Pop
```

### BGP neighbor
```
root@P1> show bgp neighbor 10.1.1.7
Peer: 10.1.1.7+63122 AS 65002  Local: 10.1.1.6+179 AS 65001
  Group: UNDERLAY-P2           Routing-Instance: master
  `Forwarding routing-instance: spoof-check`
  Type: External    State: Established    Flags: <Sync>
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: None
  Export: [ STATIC_LABEL ]
  Options: <LocalAddress AddressFamily PeerAS LocalAS Rib-group Refresh>
  Options: <GracefulShutdownRcv>
  Address families configured: inet-vpn-unicast inet-labeled-unicast
  Local Address: 10.1.1.6 Holdtime: 90 Preference: 170
  Graceful Shutdown Receiver local-preference: 0
  NLRI inet-vpn-unicast: PerPrefixLabel
  NLRI inet-labeled-unicast: PerPrefixLabel
  Local AS: 65001 Local System AS: 0
  Number of flaps: 0
  Peer ID: 1.1.1.12        Local ID: 10.1.1.6          Active Holdtime: 90
  Keepalive Interval: 30         Group index: 2    Peer index: 0    SNMP index: 1
  I/O Session Thread: bgpio-0 State: Enabled
  BFD: disabled, down
  Local Interface: ge-0/0/1.0
  NLRI for restart configured on peer: inet-vpn-unicast inet-labeled-unicast
  NLRI advertised by peer: inet-vpn-unicast inet-labeled-unicast
  NLRI for this session: inet-vpn-unicast inet-labeled-unicast
  Peer supports Refresh capability (2)
  Stale routes from peer are kept for: 300
  Peer does not support Restarter functionality
  Restart flag received from the peer: Notification
  NLRI that restart is negotiated for: inet-vpn-unicast inet-labeled-unicast
  NLRI of received end-of-rib markers: inet-vpn-unicast inet-labeled-unicast
  NLRI of all end-of-rib markers sent: inet-vpn-unicast inet-labeled-unicast
  Peer does not support LLGR Restarter functionality
  Peer supports 4 byte AS extension (peer-as 65002)
  Peer does not support Addpath
  NLRI(s) enabled for color nexthop resolution: inet-vpn-unicast inet-labeled-unicast
  Entropy label NLRI: inet-labeled-unicast
    Entropy label: No; next hop validation: Yes
    Local entropy label capability: Yes; stitching capability: Yes
  Table inet.0 Bit: 20000
    RIB State: BGP restart is complete
    Send state: in sync
    Active prefixes:              0
    Received prefixes:            0
    Accepted prefixes:            0
    Suppressed due to damping:    0
    Advertised prefixes:          1
  Table bgp.l3vpn.0 Bit: 30001
    RIB State: BGP restart is complete
    RIB State: VPN restart is complete
    Send state: in sync
    Active prefixes:              1
    Received prefixes:            1
    Accepted prefixes:            1
    Suppressed due to damping:    0
    Advertised prefixes:          1
  Last traffic (seconds): Received 23   Sent 0    Checked 197513
  Input messages:  Total 7289	Updates 3	Refreshes 0	Octets 138634
  Output messages: Total 7290	Updates 3	Refreshes 0	Octets 138719
  Output Queue[1]: 0            (inet.0, inet-labeled-unicast)
  Output Queue[2]: 0            (bgp.l3vpn.0, inet-vpn-unicast)
```

## During static label allocation 
This is applicable when advertising a static label also. The static labels are allocated on per prefix basis on BGP-LU. 
Check [here](https://ard92.github.io/2022/08/15/assigning-static-lu-lables.html)

The spoof check is done in such scenarios as well

```

root@P1> show route table spoof-check.mpls.0

spoof-check.mpls.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1000000            *[VPN/170] 2d 06:14:47
                       receive table inet.0, Pop
```
 
## References
- https://www.juniper.net/documentation/us/en/software/junos/multicast/topics/concept/anti-spoofing-support-for-mpls-labels.html
 
