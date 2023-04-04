---
layout: post
title: 6vPE on MPLS Option B scenario 
tags: junos
---

## 6vPE overview

6vPE defined in RFC 4659 and talks about offering VPN services for the IPv6 customers with the same IPv4 MPLS core.
Similar to how IPv4 routes are advertised over VPN with protocol next hop (PNH) as an IPv4 address (This can be loopback or interface depending on the design),  Protocol extensions have been made on BGP when IPv6 routes are advertised, BGP would construct a IPv4 mapped IPv6 address. Example lets say the PNH is 172.1.3.1 for VPN-v4 route. For VPNv6 route, the PNH would appear as ::ffff:172.1.3.1 i.e. `::ffff:<IPv4 route>`.

Once the PNH is present, one needs to resolve the route, else it would show up as a hidden route. In order for routes to resolve, for IPv4 VPN routes, the resolution happens through inet.3 table. If there is a route existing for the PNH in inet.3 table, BGP would resolve it. 

Similarly for IPv6 VPN routes, there needs to be an entry for the PNH in inet6.3 table. 

### Example of IPv4 VPN route with PNH resolved 
```
root@vsrx# run show route table CUST-V4.inet.0 extensive

CUST-V4.inet.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
22.1.1.1/32 (1 entry, 1 announced)
TSI:
KRT in-kernel 22.1.1.1/32 -> {indirect(262144)}
        *BGP    Preference: 170/-101
                Route Distinguisher: 200:200
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8a369c8
                Next-hop reference count: 3, key opaque handle: 0x0
                Source: 172.1.3.1
                Next hop type: Router, Next hop index: 548
                Next hop: 172.1.3.1 via ge-0/0/0.0, selected
                Label operation: Push 299904
                Label TTL action: prop-ttl
                Load balance label: Label 299904: None;
                Label element ptr: 0x8897a80
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                Protocol next hop: 172.1.3.1 <<<<< notice the PNH 
                Label operation: Push 299904
                Label TTL action: prop-ttl
                Load balance label: Label 299904: None;
                Indirect next hop: 0x7163884 262144 INH Session ID: 0
                State: <Secondary Active Int Ext ProtectionCand>
                Local AS: 65000 Peer AS: 65000
                Age: 14:33:30   Metric2: 0
                Validation State: unverified
                Task: BGP_65000.172.1.3.1
                Announcement bits (1): 0-KRT
                AS path: I  (Originator)
                Cluster list:  3.3.3.3 2.2.2.2
                Originator ID: 1.1.1.1
                Communities: target:200:200
                Import Accepted
                VPN Label: 299904
                Localpref: 100
                Router ID: 3.3.3.3
                Primary Routing Table: bgp.l3vpn.0
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: 172.1.3.1
                        Label operation: Push 299904
                        Label TTL action: prop-ttl
                        Load balance label: Label 299904: None;
                        Indirect next hop: 0x7163884 262144 INH Session ID: 0
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 172.1.3.1 via ge-0/0/0.0
                                Session Id: 0
                                172.1.3.0/30 Originating RIB: inet.3
                                  Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Interface
                                        Next hop: via ge-0/0/0.0
```

### Inet.3 table view 

Because the route `172.1.3.1` is present under inet.3 table, the VPNv4 routes are resolved correctly. Now, how do we ensure routes are present in inet.3 table ? The label distribution protocol takes care of it. For example, if you have LDP, then LDP installs the routes into inet.3. Similarly RSVP and BGP-LU. 

With BGP-LU we need to uses the knob to leak routes to inet.3, else by default Junos installs on inet.0 table.

```
root@vmx1# set protocols bgp group BGP-LU-NEIGHBOR family inet labeled-unicast rib inet.3
``` 

```
root@vsrx# run show route table inet.3

inet.3: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.1.3.0/30       *[Direct/0] 14:50:11
                    >  via ge-0/0/0.0
172.1.3.2/32       *[Local/0] 14:50:11
                       Local via ge-0/0/0.0
192.167.1.0/24     *[Direct/0] 14:50:11
                    >  via fxp0.0
192.167.1.4/32     *[Local/0] 14:50:11
                       Local via fxp0.0
```

### Example of IPv6 VPN route with PNH 

```
root@vsrx# run show route hidden extensive

inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)

inet.3: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)

CUST-V4.inet.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

mpls.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)

bgp.l3vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

CUST-V4.inet6.0: 4 destinations, 4 routes (3 active, 0 holddown, 1 hidden)
2001::1/128 (1 entry, 0 announced)
         BGP    Preference: 170/-101
                Route Distinguisher: 200:200
                Next hop type: Unusable, Next hop index: 0
                Address: 0x8a36374
                Next-hop reference count: 2, key opaque handle: 0x0
                Source: 172.1.3.1
                State: <Secondary Hidden Int Ext ProtectionCand>
                Local AS: 65000 Peer AS: 65000
                Age: 7
                Validation State: unverified
                Task: BGP_65000.172.1.3.1
                AS path: I  (Originator)
                Cluster list:  3.3.3.3 2.2.2.2
                Originator ID: 1.1.1.1
                Communities: target:200:200
                Import Accepted
                VPN Label: 299936
                Localpref: 100
                Router ID: 3.3.3.3
                Primary Routing Table: bgp.l3vpn-inet6.0
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: ::ffff:172.1.3.1
                        Label operation: Push 299936
                        Label TTL action: prop-ttl
                        Load balance label: Label 299936: None;
                        Indirect next hop: 0x0 - INH Session ID: 0
```

### Example of IPv6 VPN route resolved 
```
2001::4/128 (1 entry, 1 announced)
TSI:
KRT in-kernel 2001::4/128 -> {indirect(1048574)}
        *BGP    Preference: 170/-101
                Route Distinguisher: 200:200
                Next hop type: Indirect, Next hop index: 0
                Address: 0x77bc238
                Next-hop reference count: 3, key opaque handle: 0x0
                Source: 172.1.1.2
                Next hop type: Router, Next hop index: 584
                Next hop: 172.1.1.2 via ge-0/0/0.0, selected
                Label operation: Push 299968
                Label TTL action: prop-ttl
                Load balance label: Label 299968: None;
                Label element ptr: 0x7c38ac0
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 141
                Protocol next hop: ::ffff:172.1.1.2
                Label operation: Push 299968
                Label TTL action: prop-ttl
                Load balance label: Label 299968: None;
                Indirect next hop: 0x8519284 1048574 INH Session ID: 326
                State: <Secondary Active Int Ext ProtectionCand>
                Local AS: 65000 Peer AS: 65000
                Age: 12:45:50   Metric2: 0
                Validation State: unverified
                Task: BGP_65000.172.1.1.2
                Announcement bits (1): 0-KRT
                AS path: I  (Originator)
                Cluster list:  2.2.2.2 3.3.3.3
                Originator ID: 4.4.4.4
                Communities: target:200:200
                Import Accepted
                VPN Label: 299968
                Localpref: 100
                Router ID: 2.2.2.2
                Primary Routing Table: bgp.l3vpn-inet6.0
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: ::ffff:172.1.1.2
                        Label operation: Push 299968
                        Label TTL action: prop-ttl
                        Load balance label: Label 299968: None;
                        Indirect next hop: 0x8519284 1048574 INH Session ID: 326
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 172.1.1.2 via ge-0/0/0.0
                                Session Id: 141
                                ::ffff:172.1.1.2/128 Originating RIB: inet6.3
                                  Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Router
                                        Next hop: 172.1.1.2 via ge-0/0/0.0
                                        Session Id: 0
```

## Route resolution and resolvers when LDP/OSPF is used 
Typical deployments and examples that we find on Juniper documentations are the MPLS core is running OSPF and LDP. When  we have OSPF as IGP and LDP as a label distribution protocol, the inet.3 tables are installed with respective routes as explained in the previous sections.
In addition to that when enabled `ipv6-tunneling` , junos implicitely copies the routes from inet.3 table to inet6.3 table allowing the VPN V6 routes to be resolved. 

```
set protocols mpls ipv6-tunneling
```
However, in optionB scenarios where PE is connected directly and when no transport label is present, then protocol NH would be the directly connected interface. Let us consider the below topology to talk through it

## Topology 

![topology](/images/6vpe_topology.png)

Here vmx1 and vsrx are acting as PE . vmx2 and vmx3 are option b routers. Here the routers are connected with MP-iBGP with next hop self in order to advertise routes. Since all routers are connected over iBGP, iBGP - iBGP route advertisements will not happen unless the routers act as RR. 
Adding the cluster id will help advertise the routes. Family MPLS needs to be enabled on core facing interfaces. 

## How do we resolve in OptionB scenario ? 

Once the routes are advertised, in order for resolving IPv6 VPN routes, BGP looks at inet6.3 table. However, since the PNH received would be a mapped v4 based v6 address as explained in the above sections, the inet6.3 table needs to have the v4 mapped v6 address. 
In order to achieve this, one may end up enabling inet6 on the core facing interface either the mapped address or globally unique IPv6 address which then can be leaked using rib groups to inet6.3. The problem with this approach is, the interface would generate a NDP request and the far end router may not respond 
to the NDP request and the entry may go stale. When this happens, although from control plane things may look good, the traffic would not be forwarded due to NDP entries not being resolved. we can check this using `show ipv6 neighbours` 

```
root# run show ipv6 neighbors 

IPv6 Address                            Linklayer Address  State       Exp   Rtr  Secure  Interface               

::ffff:172.1.1.1	                 none               unreachable  3    no   no      ge-0/0/0.0              
```

A way to solve this to add a static NDP entry to force packets out but this is not a clean solution since we need to map the mac address for the static NDP . 
Example :
```
set interfaces ge-0/0/0 unit 0 family inet6 address ::ffff:172.1.1.1 ndp ::ffff:172.1.1.2 mac 2c:21:72:ec:8f:f0
```

Additionally, one may not even have IPv6 enabled on the core facing interface and would be strictly IPv4. The expectation would be resolve the mapped v4 based v6 address to recursively
resolve to the IPv4 interface and ARP accordingly so that the traffic can get forwareded 

In order to achieve that, we need to do 2 things

1. Configure rib group which leaks inet.3 -> inet6.3 
2. Configure a static route to the far end IP in inet.3 table with NH as the same IP 
3. Apply the rib group to the static route 

![rib-leak](/images/6vpe_leak.png)

By doing the above steps we would now advertise the NH IP to inet6.3 and in the process of copying to inet6.3, junos automatically adds ::ffff to the IPv4 address making it resolvable. 

one may question why not copy directly from inet.0 to inet6.3, since it is of different family types, junos will not allow to commit this type of config when applying on interface . you may see the below error 

```
root@vmx3# commit
[edit routing-options interface-routes]
  'rib-group'
    LEAK_INET_INET63: different address family ribs not permitted under interface-routes
error: configuration check-out failed
```

we also cannot leak the already leaked routes to inet.3 into inet6.3 because Junos does not leak secondary routes. Only primary routes are leaked and hence the need for a static route mentioned in point 2/

Additionally we also can skip adding ipv6-tunneling in such deployments because it doesnt bring in any change. we statically the copy the routes anyway 
## Configuration needed 
```
# Create rib groups to leak
# Can add policy to import specific routes as well which has not been shown here 
root@vsrx# show routing-options rib-groups
LEAK_INET3_INET63 {
    import-rib [ inet.3 inet6.3 ];
}

# Create static route in inet.3
root@vsrx# show routing-options rib inet.3
static {
    rib-group LEAK_INET3_INET63;
    route 172.1.1.1/32 next-hop 172.1.1.1;
}

# core facing interface. Notice no IPv6 family configured 
root@vsrx# show interfaces ge-0/0/0
unit 0 {
    family inet {
        address 172.1.1.2/30;
    }
    family mpls;
}
``` 

## Verification

```
root@vsrx# run show route table inet.3 protocol static extensive

inet.3: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
172.1.3.1/32 (1 entry, 1 announced)
        *Static Preference: 5
                Next hop type: Router, Next hop index: 0
                Address: 0x8a36be4
                Next-hop reference count: 5, key opaque handle: 0x0
                Next hop: 172.1.3.1 via ge-0/0/0.0, selected
                Session Id: 0
                State: <Active Int Ext>
                Local AS: 65000
                Age: 51:54
                Validation State: unverified
                Task: RT
                Announcement bits (3): 0-Resolve tree 3 1-Resolve tree 1 2-Resolve_IGP_FRR task
                AS path: I
                Secondary Tables: inet6.3
                Thread: junos-main


root@vsrx# run show route table inet6.3 extensive

inet6.3: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
::ffff:172.1.3.1/128 (1 entry, 1 announced)
        *Static Preference: 5
                Next hop type: Router, Next hop index: 0
                Address: 0x8a36be4
                Next-hop reference count: 5, key opaque handle: 0x0
                Next hop: 172.1.3.1 via ge-0/0/0.0, selected
                Session Id: 0
                State: <Secondary Active Int Ext>
                Local AS: 65000
                Age: 52:17
                Validation State: unverified
                Task: RT
                Announcement bits (2): 0-Resolve tree 2 1-Resolve_IGP_FRR task
                AS path: I
                Primary Routing Table: inet.3
                Thread: junos-main
```

### route tables

```
root@vsrx# run show route table CUST-V4.inet6.0

CUST-V4.inet6.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001::1/128        *[BGP/170] 00:52:42, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 172.1.3.1 via ge-0/0/0.0, Push 299936
2001::4/128        *[Direct/0] 18:52:55
                    >  via lo0.2
fe80::2aa:10f:fc31:4/128
                   *[Direct/0] 18:52:55
                    >  via lo0.2
ff02::2/128        *[INET6/0] 19:04:37
                       MultiRec
```

