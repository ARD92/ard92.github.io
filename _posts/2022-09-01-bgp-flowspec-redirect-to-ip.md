---
layout: post
title: BGP Flowspec and redirect to IP
tags: junos mx crpd
---

Junos supports BGP flowspec with various actions such as drop, redirect-to-vrf, redirect-to-ip. Flowspec routes are advertised from RRs usually where we would want to pass the filter rule to all the PEs on the edge. There are use cases where DDOS traffic need to be scrubbed and hence redirect-to-vrf would be used. In situations where no VRFs are used, we can use redirect-to-ip. Use cases where we just need to block traffic at edge, we can drop them using the `reject` action. There are several match and action conditions that are supported in Junos. Below is the 

## Match condition

```
root@PE2# set routing-options flow route test match ?
Possible completions:
  destination          Destination prefix for this traffic flow
  destination-port     Destination TCP/UDP port
  dscp                 Differentiated Services (DiffServ) code point (DSCP) (0-63)
  fragment
  icmp-code            ICMP message code
  icmp-type            ICMP message type
  packet-length        Packet length (0-65535)
  port                 Source or destination TCP/UDP port
  protocol             IP protocol value
  source               Source prefix for this traffic flow
  source-port          Source TCP/UDP port
  tcp-flags            TCP flags
```

## Action conditions

```
root@PE2# set routing-options flow route test then ?
Possible completions:
  accept               Allow traffic through
  community            Name of BGP community
  discard              Discard all traffic for this flow
  mark                 Set DSCP value for traffic that matches this flow (0..63)
  next-term            Continue the filter evaluation after matching this flow
  rate-limit           Rate in bits/sec to limit the flow traffic (0..1000000000000)
  redirect             Redirect(Tunnel) this flow's traffic to given next-hop address
  routing-instance     Redirect to instance identified via Route Target community
  sample               Sample traffic that matches this flow
```

## Extended community 

when `redirect-to-ip` is used, an extended community is passed along with the BGP flowspec route. The extended community as per IANA is 0x8008 which was asked to be deprecated as per [draft-ietf-idr-flowspec-redirect-ip-02](https://datatracker.ietf.org/doc/html/draft-ietf-idr-flowspec-redirect-ip-02)

Redirect-to-ip community has two parts addr_format = 0x0100 (for ipv4) with redirecto-to-ip-sub-type = 0xc so the type would be  0x010C

BGP flowspec uses a transitive IPv4/IPv6 address specific community. This is defined [here](https://www.iana.org/assignments/bgp-extended-communities/bgp-extended-communities.xhtml#trans-ipv4)
```
0x0c	Flow-spec Redirect to IPv4	[draft-ietf-idr-flowspec-redirect]	2016-03-22
0x000c	Flow-spec Redirect to IPv6	[draft-ietf-idr-flowspec-redirect-ip]	2016-03-22
```

## Type of redirection 

There are 3 types of redirection that is possible. 
1. redirect to vrf (RT based)
2. redirect to IP using Nexthop attribute + redirect to NH community. Here, redirection IP would be in NH field , community is 0x8008 (called as legacy redirect in junos). The community indicates the NH field is a redirect address
3. redirect to IP (not using next hop attribute) . This uses IPv4 and IPv6 address specific community and here we use (0xc) for IPv4. This is the newer method to redirect and is achieved with `family inet flow` NLRI 

### Configure the inet-flow family

This is type #3 as discussed in previous section. when looked at the extensive output of the flowspec route you would notice the community as 0x01C
```
set protocols bgp group <bgp_group> family inet flow 
```

Junos also supports the use of legacy redirect community 
```
set protocols bgp group <bgp_group> family inet flow legacy-redirect-ip-action send
set protocols bgp group <bgp_group> family inet flow legacy-redirect-ip-action receive
```

The older community was `0x08` subtype.
The copy `c` flag is ignored. if received a copy flag, it would just be propagated

## Topology considered
![topology](/images/flowspec.png)

## Configs
```
set protocols bgp group <group name> family inet flow

set routing-options flow route TEST match destination-port 899
set routing-options flow route TEST match destination 20.1.1.1/32
set routing-options flow route TEST match source 19.1.1.1/32
set routing-options flow route TEST then discard
```

## Verify
```
root@k8s-master:~# dk spe1
root@nj01emd> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 4 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       4          4          0          0          0          0
inet6.0
                       0          0          0          0          0          0
inetflow.0
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
192.168.1.2           65100          9          7       0       1          34 Establ
  inet.0: 1/1/1/0
  inetflow.0: 0/0/0/0
```

## References

- [flowspec-redirect-to-ip](https://datatracker.ietf.org/doc/html/draft-ietf-idr-flowspec-redirect-ip-02)
- [clarification of flowspec redirect community](https://www.rfc-editor.org/rfc/rfc7674)
- [dissemation-of-flowspec-rules](https://datatracker.ietf.org/doc/html/rfc5575)
- [iana-bgp-extended-community](https://www.iana.org/assignments/bgp-extended-communities/bgp-extended-communities.txt)
