---
layout: post
title: BGP graceful shutdown
tags: junos 
---

## Why do we need BGP graceful shutdown ? 
Typically in production networks during maintenance when links are brought down, routes which are advertised over this session would be withdrawn causing the traffic to blackhole until BGP reconverges to the next best available path. In order to handle this scenario gracefully, the feature of graceful-restart was introduced. In a nutshell, all we are doing is we will indicate the upstream node that the link is going to go undermaintenance and advertise with a lower local preference. So that the node can choose an alternative path and traffic would be drained. Once drained we, can issue the shutdown command 

Let us consider the topology 

```
                    1.1.1.1
                  ┌─────────┐                    ▲
                  │         │                    │
       ┌──────────┤   r1    ├─────────┐          │
       │          │         │         │          │
       │          └─────────┘         │          │
       │                              │          │
       │eBGP                     eBGP │          │
       │                              │          │
       │                              │          │
       │                              │          │
  ┌────┴────┐                    ┌────┴────┐     │ E2E traffic  1.1.1.1 -> 4.4.4.4
  │         │                    │         │     │
  │   r2    │                    │    r3   │     │
  │         │                    │         │     │
  └────┬────┘                    └────┬────┘     │
       │                              │          │
       │                              │          │
       │                              │          │
primary│eBGP                     eBGP │ secondary│
path   │           ┌─────────┐        │ path     │
       │           │         │        │          │
       └───────────┤    r4   ├────────┘          │
                   │         │                   ▼
                   └─────────┘
                      4.4.4.4
````

Here traffic from 1.1.1.1 -> 4.4.4.4 takes path r1 -> r2 -> r4 . Let us say link between r2<->r4 is going undermaintenance, traffic drop would be there until bgp converges. In order to avoid that a well known BGP community `graceful-restart` is attached to the prefixes advertised which would indicate to r2 that the link is going to go under maintance and choose the path r2->r1->r3->r4. This will drain the traffic towards the next best path. When the link is going to be shutdown, we can send a notification message such that peers would have an indication when it goes down. 

## Configuration

## State before graceful shutdown is implemented 

### On r4 
```
root@r4# run show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
192.168.3.2           65532         60         59       0       1       25:46 Establ
  inet.0: 0/0/0/0
192.168.4.2           65533         89         89       0       0       39:07 Establ
  inet.0: 0/0/0/0
```

### On r2 

```
root@r2# run show route 4.4.4.4

inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[BGP/170] 00:00:08, localpref 100
                      AS path: 65534 I, validation-state: unverified
                    >  to 192.168.3.1 via r2_r4
                    [BGP/170] 00:39:57, localpref 100
                      AS path: 65531 65533 65534 I, validation-state: unverified
                    >  to 192.168.1.1 via r2_r1
```
note that 4.4.4.4 is reachable through 2 paths, r2-> r1 and r2 -> r4. The preferred route is r2->r4 
 
### Configure BGP graceful shutdown on R4

```
root@r4# edit protocols bgp group TO_R2
graceful-shutdown {
    sender {
        local-preference 10;
    }
}
```
Once this is configured on r4 and commited, we will advertise with local pref 10 so that routes would now prefer r2->r1 

#### On r4
```
root@r4# run show route advertising-protocol bgp 192.168.3.2 extensive

inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
* 4.4.4.4/32 (1 entry, 1 announced)
 BGP group TO_R2 type External
     Nexthop: Self
     AS path: [65534] I
     Communities: graceful-shutdown
```

#### On r2 
```
root@r2# run show route 4.4.4.4

inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[BGP/170] 00:45:23, localpref 100
                      AS path: 65531 65533 65534 I, validation-state: unverified
                    >  to 192.168.1.1 via r2_r1
                    [BGP/170] 00:01:37, localpref 0
                      AS path: 65534 I, validation-state: unverified
                    >  to 192.168.3.1 via r2_r4
```
At this moment, routes are pointing to different interfaces and traffic drained gracefully. 

In order to shutdown the BGP session with a notification, we run the below and commit 

```
root@r4# show | compare
[edit protocols bgp group TO_R2]
+     shutdown {
+         notify-message "maintenance in progress";
+     }

[edit]
root@r4# commit
commit complete
```
On R2, notice BGP session is down and the only path available is the one already traffic is running on

```
root@r2# run show route 4.4.4.4

inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[BGP/170] 00:47:47, localpref 100
                      AS path: 65531 65533 65534 I, validation-state: unverified
                    >  to 192.168.1.1 via r2_r1
```
verify the BGP neighborship on r2 and notice the `Last received message`. This would be the same as advertised by r4.

```
root@r2# run show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 1
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
192.168.1.1           65531        117        121       0       0       51:13 Establ
  inet.0: 1/1/1/0
192.168.3.1           65534          2          0       0       2        1:06 Active

root@r2# run show bgp neighbor 192.168.3.1
Peer: 192.168.3.1 AS 65534     Local: 192.168.3.2 AS 65532
  Last Received Notify Msg: maintenance in progress
  Group: TO_R4                 Routing-Instance: master
  Forwarding routing-instance: master
  Type: External    State: Active         Flags: <>
  Last State: Idle          Last Event: Start
  Last Error: None
  Options: <LocalAddress AddressFamily PeerAS Refresh>
  Options: <GracefulShutdownRcv>
  Address families configured: inet-unicast
  Local Address: 192.168.3.2 Holdtime: 90 Preference: 170
  Graceful Shutdown Receiver local-preference: 0
  Number of flaps: 2
  Last flap event: RecvNotify
  Receive eBGP Origin Validation community: Reject
  Error: 'Cease' Sent: 0 Recv: 35
```

## References
- Really good [NLNOG](https://www.youtube.com/watch?v=HGGRsJ-gjI4) video which talks about graceful shutdown
- [RFC 8326](https://www.rfc-editor.org/rfc/rfc8326)
- [juniper-docs-graceful-shutdown](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/ref/statement/graceful-shutdown-edit-protocols-bgp.html)
- [juniper-docs-shutdown](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/ref/statement/graceful-shutdown-edit-protocols-bgp.html)
