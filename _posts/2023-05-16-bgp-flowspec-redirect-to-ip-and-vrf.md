---
layout: post
title: BGP flowspec with both redirect to IP and redirect to VRF actions
tags: junos 
---

BGP flowspec is a very commonly used feature for DDOS protection and redirection of traffic. Additionally one can think of it as ACLs with BGP control plane so that we can introduce blocking of a particular type of traffic based on BGP signalling which makes it very interesting from an operations perspective because we do not need to configure the node. There are various actions and we discussed in particular [redirect-to-ip](https://ard92.github.io/2022/09/01/bgp-flowspec-redirect-to-ip.html) as an action in the earlier post which signals to redirect the traffic to the tunnel endpoint (protocol next hop). 

An alternative approach would be to use redirect-to-vrf action which redirects the trafic into a VRF and the traffic can be transported accordingly. This depends on the use case. One way is to redirect the dirty traffic to a scrubber and get the clean traffic using the idea of dirty VRF and clean VRF. Each VRF would have a route target and the flowspec action would match the dirty VRF route target and dump the traffic in so that it can reach the scrubber. 

But, how about a scenario where one wants to achieve both redirect-to-ip and redirect-to-vrf as a joint action. This could be for various use cases and one common use case is performing filter based forwarding where the end point is reachable from a different VRF. 

As a firewall filter apprach one can easily solve this using the below stanza 

```
set firewall family inet filter redirect term 10 from soure-address 1.1.1.1 
set firewall family inet filter redirect term 10 then next-ip 192.168.1.1 routing-instance BAR-VRF
```

The redirect-to-ip draft covers this exact use case however from an implementation perspective there is a gap today. The BGP control plane signaling works fine however the forwarding plane would not work as expected. This post talks about an alternative way to solve the problem leveraging classful transport infrstructure on Junos 

![ct-redirect-to-ip](/images/ct_redirect_to_ip.png)

Here traffic is ingressing into foo.inet.0 and we would want to redirect to bar.inet.0 

## Signaling both redirect-to-ip and redirect-to-vrf

### View from BGP control plane perspective. This does not work on forwarding plane :) 
```
root@dcu# run show route table FOO-VRF.inetflow.0 extensive
 
FOO-VRF.inetflow.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden
99.0.0.1,98.0.0.1,proto=6,tcp-flag:01/term:1 (1 entry, 1 announced)
TSI:
KRT in dfwd;
Action(s): accept,count
        *Flow   Preference: 5
                Next hop type: Indirect, Next hop index: 0
                Address: 0x7796ae4
                Next-hop reference count: 1, key opaque handle: 0x0, non-key opaque handle: 0x0
                Next table: inet.0
                Next-hop index: 1
                Next hop: , selected
                Protocol next hop: 192.168.1.19
                Indirect next hop: 0x74eac80 - INH Session ID: 0
                State: <Active SendNhToPFE>
                Age: 1:05     Metric2: 0
                Validation State: unverified
                Task: RT Flow
                Announcement bits (1): 0-Flow
                AS path: I
                Communities: redirect-to-ip:192.168.1.19:0 redirect:999:999 <<<<<<<< note that both communities are present
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: 192.168.1.19 ResolvState: Resolved
                        Indirect next hop: 0x74eac80 - INH Session ID: 0
                        Indirect path forwarding next hops: 1
                        Next table: inet.0
                        Next-hop index: 1
                                Next hop:
                                0.0.0.0/0 Originating RIB: FOO-VRF.inet.0
                                  Node path count: 1
                                  Forwarding nexthops: 1
                                Next table: inet.0
                                Next-hop index: 1
                                        Next hop

```

## Classful transport 
Classful transport is a new way in Junos which can be used to signal intent. What I mean by that is, we can now signal color communities over BGP, RSVP, SR protocols . With this infrastructure we can also signal resolution RIBs. If we receive a route with a special community that is user defined , we can inform Junos to resolve that route from a particular routing table. you can read more on [classful-transport](https://ard92.github.io/2022/06/16/BGP-classful-transport.html). 

## Configuring resolution schemes which flowspec can leverage
### Config
```
routing-options {
    resolution {
        scheme FLOWSPEC-RES {
            resolution-ribs DIRTY-VRF.inet.0; >>>>>> Resolve flowspec route from DIRTY-VRF.inet.0
            mapping-community 100:100; >>>>> Apply resolution only to routes arriving with community 100:100 and map them to resolution ribs
        }
    }
```
This will be applied only to BGP received routes and not locally originated flowspec routes.Configuration should be on global (master instance) where flowspec NLRI is configured. The resolution is local and the configuration is local to the box.

## Captures

### Resolution schemes
```
root# run show route resolution scheme all
Resolution scheme: FLOWSPEC-RESOLUTION
  References: 1
  Mapping community: 100:100
  Resolution Tree index 8, Nodes: 6
  Policy: [__resol-schem-common-import-policy__]
  Contributing routing tables: BAR-VRF.inet.0
```

### Route to destination and redirect IP
```
root> show route 192.168.1.2 
 
inet.0: 191070 destinations, 366119 routes (56319 active, 0 holddown, 269501 hidden)
+ = Active Route, - = Last Active, * = Both
 
192.168.1.2/32     *[BGP/170] 20:44:48, localpref 100, from 10.240.0.1
                      AS path: 126 I, validation-state: unverified
                    >  to 10.185.2.188 via ae1.200, Push 128023
                    [BGP/170] 20:44:49, localpref 100, from 10.240.0.7
                      AS path: 126 I, validation-state: unverified
                    >  to 10.185.2.188 via ae1.200, Push 128023
 
BAR-VRF.inet.0: 5 destinations, 8 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
 
192.168.1.0/24     *[Direct/0] 00:06:17
                    >  via irb.11
 
FOO-VRF.inet.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
 
0.0.0.0/0          *[Static/5] 22:23:09
                       to table inet.0
```

### Flowspec route with resolution scheme community and redirect-to-ip extended community
```
FOO-VRF.inetflow.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
10.178.99/24,10.178.100.128/25,proto=1/term:1 (1 entry, 1 announced)
TSI:
KRT in dfwd;
Action(s): redirect-to-nexthop nhid: 1048575,count
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x176c929c
                Next-hop reference count: 9, key opaque handle: 0x0, non-key opaque handle: 0x0
                Source: 10.74.18.100
                Next hop type: Router, Next hop index: 670
                Next hop: 192.168.1.2 via irb.11, selected
                Session Id: 9c6
                Protocol next hop: 192.168.1.2
                Indirect next hop: 0x8cfbf08 1048575 INH Session ID: 2503
                State: <Secondary Active Int Ext SendNhToPFE>
                Local AS: 13979 Peer AS:  7018
                Age: 9:15 	Metric2: 0
                Validation State: unverified
                Task: BGP_7018_7018.10.74.18.100
                Announcement bits (1): 0-Flow
                AS path: I
                Communities: 100:100 8030:6689 target:13979:9801758 origin:2:840 redirect-to-ip:192.168.1.2:0
                Import Accepted Analyze
                Localpref: 100
                Router ID: 135.91.31.88
                Primary Routing Table: bgp.invpnflow.0
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: 192.168.1.2
                        Indirect next hop: 0x8cfbf08 1048575 INH Session ID: 2503
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 192.168.1.2 via irb.11
                                Session Id: 9c6
                                192.168.1.0/24 Originating RIB: BAR-VRF.inet.0 <<<<<< notice the redirect-vrf community resolved based on resolution schemes
                                  Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Interface
                                        Next hop: via irb.11

```

### Interface stats based on redirection
```
Interface: irb.11, Enabled, Link is Up
Flags: SNMP-Traps 0x4004000
Encapsulation: ENET2
Local statistics:                                                Current delta
  Input bytes:                      1932                                   [0]
  Output bytes:                    30314                                   [0]
  Input packets:                      23                                   [0]
  Output packets:                    631                                   [0]
Remote statistics:
  Input bytes:                     38112 (0 bps)                           [0]
  Output bytes:                     3900 (392 bps)                      [2600]
  Input packets:                     626 (0 pps)                           [0]
  Output packets:                     39 (0 pps)                          [26]
```


### Flowspec filter stats 
```
root> show firewall filter __flowspec_FOO-VRF_inet__

Filter: __flowspec_FOO-VRF_inet__
Counters:
Name                                                                            Bytes              Packets
10.178.99/24,10.178.100.128/25,proto=1                                    12262017900            122620179
10.178.99/24,10.178.100.128/25,proto=17,dstport=53                     76161379702760        1384752358232
4.33.11.66,5.22.11.66,proto=6                                                       0                    0
```

### Tcpdump on host where redirected packets arrive
```
root@ubuntu:~# tcpdump -i ens3f0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens3f0, link-type EN10MB (Ethernet), capture size 262144 bytes
18:13:16.281446 IP 10.178.100.129 > 10.178.99.1: ICMP echo request, id 13105, seq 33, length 80
18:13:18.281686 IP 10.178.100.129 > 10.178.99.1: ICMP echo request, id 13105, seq 34, length 80
18:13:20.281685 IP 10.178.100.129 > 10.178.99.1: ICMP echo request, id 13105, seq 35, length 80
18:13:22.281926 IP 10.178.100.129 > 10.178.99.1: ICMP echo request, id 13105, seq 36, length 80
```

## Advantages 

- Continue to leverage the existing flowspec controllers when both extended communities are present. 
- Simple action with just a mapping community and redirect-to-ip. No need for redirect-to-vrf community . Recursive resolution done automatically by BGP 
- Leverage pRPD functionality if controller is Junos based device for programming at scale dynamically
- Simple Configuration and is local to MX
- No need to wait for Junos implementation to support both extended communities together
- Functionality available since 21.3!! 
- BGP-CT extensions are already part of IETF Discussions (Flowspec over CT) . This uses only the CT infrastructure
  read [ietf-draft](https://datatracker.ietf.org/doc/html/draft-ietf-idr-bgp-ct-03#name-flowspec-redirect-to-ip) 
