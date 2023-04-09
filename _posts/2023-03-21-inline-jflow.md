---
layout: post
title: Inline J-Flow 
tags: junos mx
---

Service providers often monitor flows from various points. Peering, security stand point to analyze what type of flows enter the network. There are multiple ways to achieve monitoring of flows
using Junos using features such as port mirroring, sampling of packets using Jflow or sampling using inline monitoring.

Inline Jflow is used for flow monitoring and aggregating flows into an IPFIX packet (v9 / v10) which is sent to a collector. With this feature enabled, you can monitor all flows (IPv4/IPv6) ingressing/egressing a router follwing [RFC 7011](https://www.rfc-editor.org/rfc/rfc7011.html). Additionally active flow monitoring can also be configured for mpls/vpls/bridge.

More information on inline jflow available on [official documentation](https://www.juniper.net/documentation/us/en/software/junos/flow-monitoring/topics/concept/inline-sampling-overview.html)

## Some pointers to know when using inline jflow  
- Junos can perform 1-1 sampling and scales really well depening on line card
- by default template refresh rate is 10 mins
- template refresh can be seconds or packets (num of packets post which template packet sent out) .. can exist together which ever comes first, will be executed
- 1 packet can fit 4 IPv4 flows
- for IPv6 and MPLs only 2 flows can fit in a packet so that many pkts are generated
- disable template refreshes based on packet

### Configuration
```
set services flow-monitoring version-ipfix template IPFIX_V4 flow-active-timeout 30
set services flow-monitoring version-ipfix template IPFIX_V4 flow-inactive-timeout 40
set services flow-monitoring version-ipfix template IPFIX_V4 template-id 1024
set services flow-monitoring version-ipfix template IPFIX_V4 nexthop-learning enable
set services flow-monitoring version-ipfix template IPFIX_V4 template-refresh-rate packets 1
set services flow-monitoring version-ipfix template IPFIX_V4 template-refresh-rate seconds 10
set services flow-monitoring version-ipfix template IPFIX_V4 ipv4-template

set forwarding-options sampling instance SAMPLE-1 input rate 10
set forwarding-options sampling instance SAMPLE-1 family inet output flow-server 192.171.1.2 port 45000
set forwarding-options sampling instance SAMPLE-1 family inet output flow-server 192.171.1.2 version-ipfix template IPFIX_V4
set forwarding-options sampling instance SAMPLE-1 family inet output inline-jflow source-address 192.171.1.3
set forwarding-options sampling instance SAMPLE-1 family inet output inline-jflow flow-export-rate 1

set chassis fpc 0 sampling-instance SAMPLE-1

set firewall family inet filter test term 10 then count-test
set firewall family inet filter test term 10 then sample << non terminating filter action 
set firewall family inet filter test term 10 then accept << terminating filter action

set interfaces ge-0/0/0.0 family inet filter input test
set interfaces ge-0/0/0.0 family inet filter output test
```

### Verification
```
run show services accounting flow inline-jflow fpc-slot 0
  Flow information
    FPC Slot: 0
    Flow Packets: 127288835, Flow Bytes: 178366763251
    Active Flows: 0, Total Flows: 567
    Flows Exported: 476, Flow Packets Exported: 357
    Flows Inactive Timed Out: 108, Flows Active Timed Out: 459
    Total Flow Insert Count: 108

    IPv4 Flows:
    IPv4 Flow Packets: 127288835, IPv4 Flow Bytes: 178366763251
    IPv4 Active Flows: 0, IPv4 Total Flows: 567
    IPv4 Flows Exported: 476, IPv4 Flow Packets exported: 357
    IPv4 Flows Inactive Timed Out: 108, IPv4 Flows Active Timed Out: 459
    IPv4 Flow Insert Count: 108
```

### Packet format

#### Inline Jflow template packet 
![jflow-template](/images/jflow-template.png)

#### Inline Jflow flow packet 
![jflow-flow](/images/jflow-flow.png)


## How about scenarios when you want to monitor encapsulated packets ? 
In situations where one needs to analyze flows which are encapsulated in tunnels, one can rely on inline monitoring. This encapsulates flows in the datalinklayer section which can used to decode.

Take a look at [inline-monitoring](https://ard92.github.io/2022/09/12/inline-monitoring.html) blog which could be used to truncate a payload and sample a GTP packet 

you can find an inline monitor decoder [here](https://github.com/ARD92/imon-decoder) for finding flows encapsulated in GTP headers

Take a look at [port-mirroring](https://ard92.github.io/2022/06/17/packettruncation.html) blog for information around port mirroring
 
