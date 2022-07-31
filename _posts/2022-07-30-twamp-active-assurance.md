---
layout: post
title: Paragon Active Assurance & TWAMP
tags: linux junos mx
---

Juniper routers have protocols to monitor performance using RPM (Real time performance monitoring) . It utilizes active probes to monitor network performance. Service interfaces (Si-) are used where inline monitoring is enabled. RPM has multiple probe types such as ICMP, TCP, UDP. Some are supported on RE based mechanism and not entirely inline. (ex: BGP, HTTP) 

What is Jitter ? it is the difference in relative transit time between two conssecutive probes.
![jitter](/images/jitter.png){:class="img-responsive"}

D(i,j) = (Rj - Sj) - (Ri - Si) 
where S is the probe sender ,R is the probe responder and D(i,j) is the measure of jitter between probes i and j in the direction source to destination. 
The probe responder simply turns the packet around.

- Round Trip Time (RTT): (T4 - T1) - (T3-T2)
- Egress delay : (T2 - T1)
- Ingress delay: (T4 - T3)

For RE based time stamp, the value is inserted by the kernel and for the hardware based timestamp, they are taken from the packet which was filled by the PFE ukern.

## TWAMP (Two Way Active Measurement Protocol)
Open protocol for measuring two way metrics. This is written as part of `RFC 5357`

[Click here for the RFC](https://www.rfc-editor.org/rfc/rfc5357.html)

```
+------------------+                      +---------------------+
|                  |     control          |  server             |
|  control-client  +----------------------+                     |
|                  |                      |                     |
|  session-sender  +----------------------+ session reflector   |
|                  |      test            |                     |
+------------------+                      +---------------------+
```

The control and session need not be on the same system.
- *TWAMP-Control* : Used to initiate, start and stop test sessions
- *TWAMP-Test*: Used to exchange test packets between session sender and session reflector

Session sender and session reflector exchange test packets according to the TWAMP-test protocol for each active session. The control client initiates requests for TWAMP test sessions. The TCP connection gets initiated on port 862 using TWAMP control protocol. Session stop comes only from control client.
DSCP values can be set in the TCP SYN packets as well as part of various tests. 

In case of need to run TWAMP reflector/server from within a VRF, a mapping needs to happen and needs to run on a port other than `862`. 

## Configuring TWAMP reflector

```
[edit interfaces]
+   si-0/0/0 {
+       unit 0 {
+           family inet;
+       }
+       unit 10 {
+           rpm twamp-server;
+           family inet {
+               address 31.168.222.11/32;
+           }
+       }
+   }

root@R8# show | compare rollback 1
[edit chassis]
+   fpc 0 {
+       pic 0 {
+           inline-services {
+               bandwidth 1g;
+           }
+       }
+   }

[edit]
+  services {
+      rpm {
+          twamp {
+              server {
+                  authentication-mode none;
+                  port 862;
+                  client-list Client1 {
+                      address {
+                          30.1.1.2/32;
+                      }
+                  }
+              }
+          }
+      }
+  }

```
Here the address `30.1.1.2` are the test agents/client which would send the test packets. If a client list is configured only those addresses are allowed to talk to the TWAMP server. Rest would be rejected to make a connection to the port 862 (default). If the si- interface would be placed in a VRF, then the below config needs to enabled. Notice that the ports used are different than `862`

```
root@PE1> show configuration services
rpm {
    twamp {
        server {
            routing-instance-list {
                VRF1 {
                    port 900;
                }
                VRF2 {
                    port 901;
                }
                VRF3 {
                    port 902;
                }
            }
```
Here TWAMP server listens on port 900 for VRF1, 901 for VRF2 and 902 for VRF3.
 
Couple things to ensure
- enhanced-ip should be enabled under [ edit chassis fpc ]
- Although multiple units can be configured under si- interfaces only one unit can run the knob `rpm twamp-server` to act as a twamp server
    This would mean you can have only one TWAMP server/reflector per PIC.
- If we need to design where TWAMP reflectors need to be present per VRF , we can use VRF leaking 
    - Create a common dedicated TWAMP VRF where TWAMP server is reachable and leak routes to other VRFs so that packets are reachable
    - Create si interface per PIC and place them individually in VRFs. THis is not scalable since there would be limitations based on line cards. If the number of test VRFs are limited this is an option that can be considered since its easier to manage from config perspective.
    -  Let TWAMP server run on global 862 port and leak routes to the VRFs. Here things act on global table but follows design of option 1 closely.

The above config is for TWAMP full. In case TWAMP lite needs to used, 
```
set services rpm twamp client control-connection c1 control-type light 	# Client side
set services rpm twamp server light port <port-number> 		# Server side
```

## TWAMP light 
- Helps in larger scale deployments of servers 
- removes control plane TCP session. So responder performance is improved and allows to rapidly respond. 
- Stateless version of TWAMP full where test params are predefined instead of negotiated.

## Scale
### TWAMP full
- 500 sessions
- 2 sockets needed (1 for control and 1 for sending probes)
- RE is responsible for managing concurrent sessions where Junos kernel scale comes into play. Around 1k concurrent connections.

## TWAMP light
- 1000 concurrent light sessions for server/client

## Paragon active assurance 
Paragon active assurance helps in generating active L2-L7 traffic generation and helps in testing, monitoring of networks. Can be used as a SaaS or Onprem environments. These data plane metrics help in validating/understanding/resolving underlying quality issues. Consists of 2 parts
1. Control center : The control center hosts the GUI. The tests/config will be controlled from here.  
2. Test agents:
    - Available in VM or container form factor 
    - Can be hosted on on prem/azure/AWS

These tests can be automated using YANG/REST APIs. 

### Type of tests 
Variety of tests are available.
1. Stateful TCP, UDP
2. IPTV and OTT Video
3. Voice
4. Internet performance
5. Remote packet inspection
6. Wifi

### Setup
- Follow the instructions at https://www.juniper.net/documentation/us/en/software/active-assurance3.1/paa-install/topics/concept/introduction.html
- It is a 1-1 follow, do not miss any steps 
- If the VM is inside the server with no direct reachability
  SSH tunnel to access the port 443
    ```
    ssh -L 8845:192.168.2.3:443 root@10.85.47.161
    ```

### Container based test agents

#### Load image
```
docker load –i paa-test-agent-application-container_3.1.0.19_amd64.tar.gz
```

#### Register the test agent
```
docker run --rm -v $(pwd):/config paa/test-agent-application:3.1.0.19 \
         register --config /config/agent.conf \
                  --ncc-host 192.168.2.3 \
                  --account paa \
                  --api-token ZtbAJuxobha99kiOHCa4LaNmN4P6ZMVpPfe68sgC \
                  --name paa-agent1 \
                  --ssl-no-check
```
- ncc-host: The host ip where control center is running 
- account: active assurance account used. Can be retrieved from `ncc user-list`
- api-token : This can be retrieved under users in GUI. Or as an alternative you can use username and password.

```
root@paa-server:~# ncc user-list
Users list

xyz@abc.net
Access: active
Accounts:
    Name: paa (paragon active assurance)
    Permission: admin
    Status: owner

confd@netrounds.com
Access: active
Accounts:
```

#### Start the container
```
docker run -d --privileged -v $(pwd):/config paa/test-agent-application:3.1.0.19 --config /config/agent.conf
```

This will start the container and have it registered on the control center. It must now be visible.

#### Add secondary interfaces to the container
you can add any number of interfaces to your docker container using veth interfaces. I use the script to achieve the same. The script can be found [here]()

```
python veth_connect.py -action cveth -name eth1:R88 << create a veth pair
veth with name eth1 and peer-name R88 created

python veth_connect.py -action conn -name eth1 -ip 30.1.1.2/30 -container 643a4bf43dce
entered here
Above command well move one interface into the TA container

brctl addif R8_CE R88 << will move the other end of veth into bridge R8_CE where VM interface is connected
```

_Note: Note: ensure TA is reachable to the TWAMP reflector IP. You will have to add routes within the TA container accordingly if its network is not reachable_

![TA](/images/TA.png){:class="img-responsive"}

#### Configure TWAMP reflector in settings in GUI

Go to account and settings --> TWAMP --> Add new reflector
![reflector](/images/TWAMP_reflector.png){:class="img-responsive"}

#### Create a new test
![newtest](/images/paa_test.png){:class="img-responsive"}

notice the traffic on ingress interface on the vMX 

```
 Delay: 2/0/5
Interface: ge-0/0/2, Enabled, Link is Up
Encapsulation: Ethernet, Speed: 1000mbps
Traffic statistics:                                              Current delta
  Input bytes:                  85203604 (986368 bps)                 [764888]
  Output bytes:                 16352792 (986368 bps)                 [763048]
  Input packets:                   61798 (89 pps)                        [638]
  Output packets:                  13010 (89 pps)                        [608]
Error statistics:
  Input errors:                        0                                   [0]
  Input drops:                         0                                   [0]
  Input framing errors:                0                                   [0]
  Carrier transitions:                 1                                   [0]
  Output errors:                       0                                   [0]
  Output drops:                        0                                   [0]
```

![results](/images/test_tcp.png){:class="img-responsive"}
