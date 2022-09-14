---
layout: post
title: Inline monitoring 
author: Aravind
tags: mx
---

There are various ways to mirror a packet and truncate them. Read [here] for more information on how to mirror and truncate packets on MX. There might be requirements from SPs where they would like to capture additional information on meta data. In such cases inline monitoring is helpful. 

## Configuration
inline monitoring is applied as per firewall filter action.

### Step1: Define inline monitor template
```
root@rtr01# show services
inline-monitoring {
    template {
        GTP-1 {
            template-refresh-rate 10;
            observation-domain-id 1;
            primary-data-record-fields {
                direction;
                datalink-frame-size;
            }
        }
    }
```

### Step2: Define inline monitor params 
define instance `GTP-1`
```
root@rtr01# show services
inline-monitoring {
    template {
        GTP-1 {
            template-refresh-rate 10;
            observation-domain-id 1;
            inactive: primary-data-record-fields {
                direction;
                datalink-frame-size;
            }
        }
    }
    instance {
        GTP-1 {
            template-name GTP-1;
            inactive: maximum-clip-length 65;
            collector {
                COLLECTOR-1 {
                    source-address 20.1.1.2;
                    destination-address 20.1.1.1;
                    destination-port 20000;
                    sampling-rate 1;
                }
            }
        }
    }
}
```

### Step3: Create firewall filter with action inline monitor 
```
root@rtr01# show firewall family inet filter GTPv4
term teid-1000 {
    from {
        gtp-header {
            gtp-teid 1000;
            ipv4 {
                protocol udp;
                source-port 1000;
                destination-port 2000;
                source-address {
                    1.1.1.1/32;
                }
                destination-address {
                    2.2.2.2/32;
                }
            }
        }
    }
    then {
        count GTP-1;
        inline-monitoring-instance GTP-1;
        accept;
    }
}
```
### Step4: Map the filter to the IFL
```
root@rtr01# show interfaces ge-0/0/0
passive-monitor-mode;
unit 0 {
    family inet {
        filter {
            input GTPv4;
        }
        address 10.1.1.1/30;
    }
}
```

## Scaling
- 16 inline-monitoring instaces configurable 
    - each instance can have different clip lengths. so 16 different clip lengths
- each instance can support upto 4 IPv4 collectors
- Total of 64 possible collectors. 
- This is part of filter configuration where terminating action is inline monitoring instance 
- each term can be a different instance 

## Additional Notes 
- full packet isnt carried in the MX when using inline-monitor. Gets clipped at max 126B. 
- timestamps on the IPFIX packet is 1s granularity (MX platforms)
- when capturing on collector it is better to capture until template refresh time configured. 
    - The first packet on collector is the template packet.
    - if this is not captured, wireshark may not be able to decode the remainder since the template doesnt exist.

## Verify

### Generate packet
generate packets using a test set. I used [go-packet-crafter](https://github.com/ARD92/go-packet-crafter)
```
./go-packet-gen -m B6:44:66:5E:33:7C -M 02:aa:01:10:01:00 -S 1.1.1.1 -D 2.2.2.2 -t udp -i pkt1 -H true -s 1000 -d 2000

============= Hex packet crafted ===========

02aa01100100b644665e337c080045000025000000000011b4c3010101010202020203e807d00011c96f676f7061796c6f6164000000000000000000
=================================================

./go-packet-gen -m B6:44:66:5E:33:7C -M 02:aa:01:10:01:00 -S 10.1.1.2 -D 10.1.1.1 -t udp -i pkt1 -H true -s 1000 -d 2152 -T 1000 -x 02aa01100100b644665e337c080045000025000000000011b4c3010101010202020203e807d00011c96f676f7061796c6f-18446744073709551615 -n 13
```

### Verify filter counters 
```
root@sims# run show firewall

Filter: __default_bpdu_filter__

Filter: GTPv4
Counters:
Name                                                Bytes              Packets
GTP-1                                                 949                   13
```

### Verify packet capture

#### Template packet
![packetcapture](/images/imon_template_packet.png)
#### Subsequent packets
![packetcapture](/images/inlinemon_pcap.png)

