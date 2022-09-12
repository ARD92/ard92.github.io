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
            inactive: primary-data-record-fields {
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
- each instance can support upto 4 IPv4 collectors
- Total of 64 possible collectors. 
- This is part of filter configuration where terminating action is inline monitoring instance 
- each term can be a different instance 

## Additional Notes 
