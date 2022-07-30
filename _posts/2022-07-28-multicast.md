---
layout: post
title: L2 and L3 Multicast 
tags: mx junos
---

*WIP*

Multicast has been around for a while and is used when data has to be transmitted to multiple receivers simulataneously. There are several applications for it such as stock tickers, video distributions, IPTV, multiplayer gaming etc.
The advantagess of multicast include
- Reduced resource utilization controlling network bandwidth reducing server and router loads
- Enhanced scalability : Network utilization independent of receivers
- Deterministic performance : Subscriber 1 or subscriber 1000, both have same performance 
- Lower TCO due to the above 
- AMT needed in case of in-between unicast islands or some sort of overlays (ex: Tunnel mcast over GRE )

Multicast can be enabled on L2 or L3 networks. L3 networks typically include PIM (Protocol Independent Multicast) for ASM (Any source mutlicast) and SSM (source Specific Multicast).
ASM has been complex since multiple sources are involved. SSM has been relatively simpler and would be preferred if a single source exist and we need to distribute to multiple receivers. 

The bigger problem of multicast is every network has to be enabled with multicast else they would not work. So if there are unicast islands then we need to leverage technologies such as AMT (Automatic Multicast Tunneling). This helps in tunneling multicast packets over unicast networks using UDP tunnels.

## Layer 2 Multicast 

### IGMP 

## Layer 3 Multicast

### PIM ASM

#### Dense Mode

##### Rendevous point

#### Sparse mode

### PIM SSM 

