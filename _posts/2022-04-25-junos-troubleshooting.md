---
layout: post
title: Junos troubleshooting commands
tags: junos 
---

This is a WIP post
 
## List of troubleshooting commands on Junos

The below nomenclature is used to write this document 
Optional:  [ ] 
Input param: < >

### using traceoptions
traceoptions can be enabled in various hierarchies specific to each protocol. This helps in immediate identification based on the type of the issue.
```
set <protocol> traceoptions flag <all/specific.. hit "?" to display the available options>
set <protocol> traceoptions file <name> size <size. example 100m> 
```
### Generic
- show route <prefix>
- show route <prefix> table <table name>
- show route protocol <protocol name> table <table name>
- ping <dest IP> source <source IP> [rapid count <num>] [routing-instance <instance-name>]
- traceroute <dest IP> [routing-instance <instance-name>]
- show interfaces <name> [detail/extensive/terse]
- show route hidden table inet.0
- show route summary 
- monitor interface <name>
- show log messages
- show chassis alarms
- show chassis hardware detail
- show chassis fpc


### BGP 
- show bgp summary
- show bgp neighbor <neighbor> 
- show route receive protocol bgp <neighbor> [extensive/detail/expanded-nh/hidden]
- show route advertising procotol bgp <neighbor> [extensive/detail/hidden]

### OSPF 
- show ospf route
- show ospf database [summary/detail/summary]
- show ospf database area <num> <nssa/opaque-area/network/link-local/router/lsa-id> [detail/extensive]
- show ospf neighbor <IP> [detail/extensive/brief]
- show ospf interface [detail]
- show ospf statistics

### Forwarding table
- show route forwarding-table destination <Dest IP>

----------------------------------------------------
The below are services related. 

## Services 

### Flows
- show security flow session
- show security flow status
- show security flow statistics
- show security policy 
- show log messages [ in case logging is enabled ]

### IPSEC on SRX/vSRX
- show security ike security-association
- show security ike security-association index <#> detail
- show security ipsec security-association
- show security ipsec security-association index <#> detail
- show security ipsec statistics
- show security ipsec statistics index <#>
- show security ipsec next-hop-tunnels
- monitor interface st0.x
- show interfaces extensive st0.x
- show security flow session tunnel
- show route
- show security pki local-cert detail
- show security pki ca-cert detail
- show security pki crl detail

https://supportportal.juniper.net/s/article/SRX-Data-Collection-Checklist-Logs-data-to-collect-for-troubleshooting?language=en_US#IpsecRouteBased

-----------------------------------------------------
The below is WIP and will be added shortly.

### IPSEC on MS-MPC

### IPSEC on SPC3 

### Inline Jflow

### CGNAT

### Stateful Firewall

### ISIS

### MPLS-VPNs

### RSVP 

### LDP 

### MPLS

### Segment routing

### Firewall filters
