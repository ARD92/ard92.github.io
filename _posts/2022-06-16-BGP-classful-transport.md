---
layout: post
title: BGP Classful Transport 
tags: junos juniper mx crpd
---

There are various ways out there which talks about network slicing and offering SLAs for a particular slice. The concept of color has been used in various ways to indicate intent for a path to be routed. Segment routing (SRTE) has a color attribute to indicate the path, Flex algo which can indicate color for an IGP slice instead of TE. These are exist within an AS domain . For inter-AS scenarios, there is FLEXS which uses BGP-LS or a controller based approach for inter-AS paths which offer E2E SLA defined path. Transport class is one such approach which can be used to carry SLA information (Color) within or across AS boundaries. BGP-LU by itself doesn't carry color attributes, so by enabling this new NLRI i.e. family transport we can extend the color . Additionally it also brings in the concept of color/intent to RSVP-TE LSPs which natively does not contain the color attribute. One does not need to expose the internal topology of a domain to another domain. Each domain is free to make its own choice of underlying transport. i.e. SR-TE, RSVP-TE, Flex-algo. By using CT, it can act as a glue across domains.

The technology itself is very similar to BGP-LU which is well understood . Can easily scale by just having labeled BGP and not IGP between domains. BGP-CT uses (egress IP, color).

A new SAFI has been allocated. SAFI 76 (family inet transport) which is used to carry color information. A new type of route-target (transport-target:0:100) Here 100 is color. 

## Why do we need BGP-CT ? 
- Within an intra-AS domain, there would be a need of having tunnels with different TE characteristics.
- Sometimes these tunnels might be extended to inter-AS scenarios to preserve characteristics to offer E2E TE services
- Different service routes (L3VPN, L2VPN, EVPN, IPV6…) may want to use a particular tunnel with specific characteristics
- Service routes must be agnostic of transport layer (RSVP-TE, SRTE, Flex Algo, IP tunnels .. ) and help resolve these routes into the respective transport layer 

### Benefits 
- Completely Distributed
- Scalable
- Use it for inter-AS as well as intra-AS deployments 
- Service and transport layers are decoupled from each other. Can interop with existing transport and yet offer TE characteristics 
- Independent sites can co-work .i.e RSVP-TE domains and SR-TE domains can work across using BGP-CT and BGP-LU
- Easy to Migrate to BGP-CT . Just need to enable a new family (family bgp-transport)
- works on physical and virtual platforms. Can extend to VMs/computes and there by we can extend intent all the way to application servers.
 
BGP-CT follows RFC8212, which helps in avoiding unintended propagation of transport-routes across EBGP boundaries and hence an import policy is needed on ASBR, else the route would be rejected as ”rejected by import policy” and show up as hidden route 
 
## Topology
Let us consider the below topology to try out BGP-CT
![topology](/images/ct-topology.png)

you can easily bring up this topology using the [crpd-topology-builder](https://github.com/ARD92/crpd-topology-builder). The topology file is [here](https://github.com/ARD92/crpd-topology-builder/blob/master/topologies/bgp-ct.yml). The respective configs are present [here](https://github.com/ARD92/crpd-topology-builder/tree/master/configs/bgp-ct/)

### Dataplane 
Multiple inet.3 would be created for resolution purposes. bgp.transport.3 is used to advertise the transport routes over BGP.

![topology](/images/ct-dataplane.png)

## Fallback to inet.3 
BGP-CT offers fall back mechanisms where we can fall back to BGP-LU if the transport class is unavailable. On the other hand, if there is a requirement of a strict SLA, the fall back option can be none and traffic can be dropped.
 
```
root@pe1# set routing-options transport-class name slice-1 fallback ?
Possible completions:
+ apply-groups         Groups from which to inherit configuration data
+ apply-groups-except  Don't inherit configuration data from these groups
  none                 Resolve associated service-routes using this transport-class tunnels only
```

## BGP-Flowspec on transport class ? 
One can run flowspec on top of the transport class in order to offer SLA for a specific set of traffic which needs redirection. Flowspec will resolve over color.inet.3 table instead of inet.3 table. There by traffic would resolve over the transport class having the defined SLA. 

### RR Config
```
routing-options {
    autonomous-system 65400;
    flow {
        route TEST-REDIRECT {
            match source 10.10.100.1/32;
            then {
                community COLOR-500; << indicates color on which flowspec route to be resolved. 
                redirect 6.6.6.6;
            }
        }
    }
    static {
        route 0.0.0.0/0 next-hop 172.16.60.1;
    }
}
protocols {
    bgp {
        group TEST-CT-REDIRECT {
            type internal;
            neighbor 172.16.60.1 {
                family inet {
                    unicast;
                    flow;
                }
            }
        }
    }
}
```

### Configuration needed for BGP-Classful transport 
1. enabling NLRI along with family labeled-unicast 
2. auto-creation of transport classes upon receipt of a color community 
3. Add an export policy to advertise routes with correct color community so BGP can carry it and on the receiving router, resolve it from correct Transport class

```
set routing-options transport-class auto-create

set protocols bgp group <GROUP_NAME> family inet labeled-unicast
set protocols bgp group <GROUP_NAME> family inet transport
```

## Captures

### Viewing transport classes
```
root@R4# run show routing transport-class all
Transport Class: junos-tc-500  Configured name: TC-500
  Color: 500, References: 2
  Transport Endpoints: IPv4 1  IPv6 0
  Mapping community: color:0:500
  Route Target: transport-target:0:500
  Routing instance: junos-rti-tc-500

Transport Class: junos-tc-600  Configured name: TC-600
  Color: 600, References: 1
  Transport Endpoints: IPv4 0  IPv6 0
  Mapping community: color:0:600
  Route Target: transport-target:0:600
  Routing instance: junos-rti-tc-600
```

### Viewing BGP routes 
```
root@R4# run show route table bgp.transport.3 extensive

bgp.transport.3: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
4.4.4.4:12:6.6.6.6/96 (1 entry, 1 announced)
TSI:
Page 0 idx 0, (group AS65402 type External) Type 1 val 0xa3ac7a8 (adv_entry)
   Advertised metrics:
     Flags: Nexthop Change
     Nexthop: Self
     MED: 1
     AS path: [65401] I
     Communities: transport-target:0:500
     Label: 72
    Advertise: 00000001
Path 4.4.4.4:12:6.6.6.6
Vector len 4.  Val: 0
        *SPRING-TE Preference: 8
                Next hop type: Indirect, Next hop index: 0
                Address: 0x77bd5a0
                Next-hop reference count: 4, key opaque handle: 0x0
                Next hop type: Router, Next hop index: 0
                Next hop: 172.16.30.2 via ge-0/0/1.0
                Label operation: Push 60006
                Label TTL action: prop-ttl
                Load balance label: Label 60006: None;
                Label element ptr: 0x7cca998
                Label parent element ptr: 0x0
                Label element references: 10
                Label element child references: 6
                Label element lsp id: 0
                Session Id: 0
                Next hop: 172.16.30.6 via ge-0/0/2.0, selected
                Label operation: Push 60006
Label TTL action: prop-ttl
                Load balance label: Label 60006: None;
                Label element ptr: 0x7cca998
                Label parent element ptr: 0x0
                Label element references: 10
                Label element child references: 6
                Label element lsp id: 0
                Session Id: 0
                Protocol next hop: 60006
                Composite next hop: 0x6f620b0 - INH Session ID: 0
                Indirect next hop: 0x711f6cc - INH Session ID: 0 Weight 0x1
                State: <Secondary Active Int>
                Local AS: 65401
                Age: 1:17:06 	Metric: 1 	Metric2: 126
                Validation State: unverified
                Task: SPRING-TE
                Announcement bits (1): 1-BGP_RT_Background
                AS path: I
                Communities: transport-target:0:500
		SRTE Policy State:
                   SR Preference/Override: 100/100
                   Tunnel Source: Static configuration
                Primary Routing Table: junos-rti-tc-500.inet.3
                Thread: junos-main
```

## Validate
Try this using cRPD or vMX. 

- use [crpd-topology-builder](https://github.com/ARD92/crpd-topology-builder/tree/master/topologies/bgp-ct) for crpd setup
- use [vm-topology-builder](https://github.com/ARD92/vm-topology-builder/tree/main/topologies/INTER-AS-BGP-CT) for vmx based setup 

necessary configs are present in the same link.

## References 
- [draft-classful-transport](https://www.ietf.org/archive/id/draft-kaliraj-idr-bgp-classful-transport-planes-17.txt)

