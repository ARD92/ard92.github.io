---
layout: post
title: BGP add path 
tags: crpd junos
---

When bgp best path selection occurs, it only chooses one path. If there is a need to advertise multiple paths for the same prefix, then BGP introduces a new capability called add-path which will advertise all the paths to a particular prefix. The capability exchange occurs in the BGP OPEN message and the path count indicates how many add paths can be sent. Additionally you can signal how many one would want to receive as well.

## Topology considered 
```
                                            <- 1.1.1.1

                                           ┌─────────┐
                                           │         │
                         ┌────ibgp─────────┤   PE2   ├───────ebgp──────┐
             1.1.1.1:PE1 │                 └─────────┘                 │
             1.1.1.1:PE2 │                                             │
                         │                                         ┌───┴──────┐
 ┌───────┐          ┌────┴───┐                                     │          │
 │       ├───ibgp───┤  RR 1  │                                     │   CE     │
 │  PE1  │          │        │                                     │          │
 └───────┘          └──┬─┬───┘                                     └───┬──────┘
                       │ │                                             │   1.1.1.1
                       │ │                                             │
                       │ │                  ┌──────────┐               │
                       │ └───ibgp───────────┤          │               │
                       │                    │   PE3    ├──────ebgp─────┘
                      ibgp                  └──────────┘
                       listener
                       │                    <- 1.1.1.1
                     ┌─┴──────────┐
                     │   monitor  │
                     └────────────┘

```

## Building topology
use crpd-topology-builder, found [here](https://github.com/ARD92/crpd-topology-builder) and use the `add-path.yml` topology file 

```
python topo_builder/topo_builder.py -a create -t topologies/add_path.yml
```
### Config router
Follow the below for all containers 
```
python topo_builder/topo_builder.py -a config -t topologies/add_path.yml -cfg backup_pe1.txt -c pe1
...
```

## Without Addpath 

- 1.1.1.1 advertised from CE towards PE2 and PE3 
- PE2 and PE3 advertise 1.1.1.1 to RR 
- RR advertises only best path to PE1 
- PE1 sees only one route to 1.1.1.1

### RR sees 2 paths 
```
root@rr# run show route 1.1.1.1

inet.0: 17 destinations, 18 routes (17 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.1.1/32         *[BGP/170] 00:00:18, localpref 100, from 100.1.1.2
                      AS path: 10000 I, validation-state: unverified
                    >  to 192.168.2.1 via rr_pe2
                    [BGP/170] 00:00:18, localpref 100, from 100.1.1.3
                      AS path: 10000 I, validation-state: unverified
                    >  to 192.168.3.1 via rr_pe3
```

### RR advertises only one best path and PE1 sees only 1 path

```
root@rr# run show route advertising-protocol bgp 100.1.1.1

inet.0: 17 destinations, 18 routes (17 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 1.1.1.1/32              100.1.1.2                    100        10000 I


root@pe1# run show route 1.1.1.1

inet.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.1.1/32         *[BGP/170] 00:00:58, localpref 100, from 100.100.100.100
                      AS path: 10000 I, validation-state: unverified
                    >  to 192.168.1.2 via pe1_rr

```

## With Addpath 

- 1.1.1.1 advertised from CE towards PE2 and PE3 
- PE2 and PE3 advertise 1.1.1.1 to RR 
- RR enabled with add-path and path-count 2 with send and receive options 
- RR advertises 2 paths to PE1 for 1.1.1.1
- PE1 configured with add-path receive 
- PE1 sees 2  paths for 1.1.1.1

### Configuration

#### RR add-path send and receive
```
root@rr# show protocols bgp group RR
type internal;
local-address 100.100.100.100;
family inet {
    unicast {
        add-path {
            receive;
            send {
                path-count 2;
            }
        }
    }
}
cluster 100.100.100.100;
neighbor 100.1.1.1;
neighbor 100.1.1.2;
neighbor 100.1.1.3;
neighbor 101.101.101.101;
```

#### PE1 add-path receive 
```
root@pe1# show protocols bgp
group RR {
    type internal;
    family inet {
        unicast {
            add-path {
                receive;
            }
        }
    }
    neighbor 100.100.100.100 {
        local-address 100.1.1.1;
    }
}
```

### Outputs
```
root@rr# run show route 1.1.1.1

inet.0: 17 destinations, 18 routes (17 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.1.1/32         *[BGP/170] 00:11:25, localpref 100, from 100.1.1.2
                      AS path: 10000 I, validation-state: unverified
                    >  to 192.168.2.1 via rr_pe2
                    [BGP/170] 00:10:27, localpref 100, from 100.1.1.3
                      AS path: 10000 I, validation-state: unverified
                    >  to 192.168.3.1 via rr_pe3


root@rr# run show route advertising-protocol bgp 100.1.1.1

inet.0: 17 destinations, 18 routes (17 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 1.1.1.1/32              100.1.1.2                    100        10000 I
                          100.1.1.3                    100        10000 I

root@rr# run show route advertising-protocol bgp 100.1.1.1 extensive

inet.0: 17 destinations, 18 routes (17 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (2 entries, 2 announced)
 BGP group RR type Internal
     Nexthop: 100.1.1.2
     Localpref: 100
     AS path: [65001] 10000 I
     Cluster ID: 100.100.100.100
     Originator ID: 100.1.1.2
     Addpath Path ID: 1
 BGP group RR type Internal
     Nexthop: 100.1.1.3
     Localpref: 100
     AS path: [65001] 10000 I
     Cluster ID: 100.100.100.100
     Originator ID: 100.1.1.3
     Addpath Path ID: 2

```

#### As seen from PE1
```
root@pe1# run show route receive-protocol bgp 100.100.100.100 extensive

inet.0: 14 destinations, 15 routes (14 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (2 entries, 1 announced)
     Accepted
     Nexthop: 100.1.1.2
     Localpref: 100
     AS path: 10000 I  (Originator)
     Cluster list:  100.100.100.100
     Originator ID: 100.1.1.2
     Addpath Path ID: 1
     Accepted
     Nexthop: 100.1.1.3
     Localpref: 100
     AS path: 10000 I  (Originator)
     Cluster list:  100.100.100.100
     Originator ID: 100.1.1.3
     Addpath Path ID: 2

root@pe1# run show route 1.1.1.1 extensive

inet.0: 14 destinations, 15 routes (14 active, 0 holddown, 0 hidden)
1.1.1.1/32 (2 entries, 1 announced)
TSI:
KRT-MFS Advt. to KRT-Netlink
KRT-Netlink in-kernel 1.1.1.1/32 -> {indirect(-)}
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x5c31e5d1379c
                Next-hop reference count: 2
                Kernel Table Id: 0
                Source: 100.100.100.100
                Next hop type: Router, Next hop index: 0
                Next hop: 192.168.1.2 via pe1_rr, selected
                Session Id: 0
                Protocol next hop: 100.1.1.2
                Indirect next hop: 0x5c31e2d79a88 - INH Session ID: 0
                State: <Active Int Ext>
                Local AS: 65001 Peer AS: 65001
                Age: 13:02      Metric2: 2
                Validation State: unverified
                Task: BGP_65001.100.100.100.100
                Announcement bits (3): 1-KRT MFS 2-KRT 6-Resolve tree 1
                AS path: 10000 I  (Originator)
                Cluster list:  100.100.100.100
                Originator ID: 100.1.1.2
                Accepted
                Localpref: 100
                Router ID: 100.100.100.100
                Addpath Path ID: 1
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: 100.1.1.2 Metric: 2 ResolvState: Resolved
                        Indirect next hop: 0x5c31e2d79a88 - INH Session ID: 0
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 192.168.1.2 via pe1_rr
                                Session Id: 0
                                100.1.1.2/32 Originating RIB: inet.0
                                  Metric: 2 Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Router
                                        Next hop: 192.168.1.2 via pe1_rr
                                        Session Id: 0
         BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x5c31e5d15c5c
                Next-hop reference count: 1
                Kernel Table Id: 0
                Source: 100.100.100.100
                Next hop type: Router, Next hop index: 0
                Next hop: 192.168.1.2 via pe1_rr, selected
                Session Id: 0
                Protocol next hop: 100.1.1.3
                Indirect next hop: 0x5c31e2d7bb08 - INH Session ID: 0
                State: <NotBest Int Ext Changed>
                Inactive reason: Not Best in its group - Router ID
                Local AS: 65001 Peer AS: 65001
                Age: 12:04      Metric2: 2
                Validation State: unverified
                Task: BGP_65001.100.100.100.100
                AS path: 10000 I  (Originator)
                Cluster list:  100.100.100.100
                Originator ID: 100.1.1.3
                Accepted
                Localpref: 100
                Router ID: 100.100.100.100
                Addpath Path ID: 2
                Thread: junos-main
                Indirect next hops: 1
                        Protocol next hop: 100.1.1.3 Metric: 2 ResolvState: Resolved
                        Indirect next hop: 0x5c31e2d7bb08 - INH Session ID: 0
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 192.168.1.2 via pe1_rr
                                Session Id: 0
                                100.1.1.3/32 Originating RIB: inet.0
                                  Metric: 2 Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Router
                                        Next hop: 192.168.1.2 via pe1_rr
                                        Session Id: 0

```

## References
- [RFC 7911](https://datatracker.ietf.org/doc/html/rfc7911)
- [juniper docs](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/bgp/topics/example/bgp-add-path.html)
