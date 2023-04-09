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

#### Capture using tshark 

##### Start tshark with decode CFLOW
```
/app # tshark -i pkt2 -d udp.port==1000,cflow -V src 20.1.1.2
```

##### Capture
```
Frame 6: 86 bytes on wire (688 bits), 86 bytes captured (688 bits) on interface pkt2, id 0
    Interface id: 0 (pkt2)
        Interface name: pkt2
    Encapsulation type: Ethernet (1)
    Arrival Time: Sep 19, 2022 16:41:45.982407681 UTC
    [Time shift for this packet: 0.000000000 seconds]
    Epoch Time: 1663605705.982407681 seconds
    [Time delta from previous captured frame: 10.005030109 seconds]
    [Time delta from previous displayed frame: 10.005030109 seconds]
    [Time since reference or first frame: 32.096940181 seconds]
    Frame Number: 6
    Frame Length: 86 bytes (688 bits)
    Capture Length: 86 bytes (688 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:udp:cflow]
Ethernet II, Src: 02:aa:01:10:01:01 (02:aa:01:10:01:01), Dst: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
    Destination: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
        Address: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 02:aa:01:10:01:01 (02:aa:01:10:01:01)
        Address: 02:aa:01:10:01:01 (02:aa:01:10:01:01)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 20.1.1.2, Dst: 20.1.1.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 72
    Identification: 0x1c88 (7304)
    Flags: 0x00
        0... .... = Reserved bit: Not set
        .0.. .... = Don't fragment: Not set
        ..0. .... = More fragments: Not set
    Fragment Offset: 0
    Time to Live: 250
    Protocol: UDP (17)
    Header Checksum: 0x7a18 [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 20.1.1.2
    Destination Address: 20.1.1.1
User Datagram Protocol, Src Port: 50151, Dst Port: 1000
    Source Port: 50151
    Destination Port: 1000
    Length: 52
    [Checksum: [missing]]
    [Checksum Status: Not present]
    [Stream index: 0]
    [Timestamps]
        [Time since first frame: 32.096940181 seconds]
        [Time since previous frame: 10.005030109 seconds]
    UDP payload (44 bytes)
Cisco NetFlow/IPFIX
    Version: 10
    Length: 44
    Timestamp: Sep 19, 2022 16:41:45.000000000 UTC
        ExportTime: 1663605705
    FlowSequence: 6
    Observation Domain Id: 16842752
    Set 1 [id=2] (Data Template): 1024
        FlowSet Id: Data Template (V10 [IPFIX]) (2)
        FlowSet Length: 28
        Template (Id = 1024, Count = 4)
            Template Id: 1024
            Field Count: 4
            Field (1/4): 137 [pen: Juniper Networks, Inc.]
                1... .... .... .... = Pen provided: Yes
                .000 0000 1000 1001 = Type: 137 [pen: Juniper Networks, Inc.]
                Length: 4
                PEN: Juniper Networks, Inc. (2636)
            Field (2/4): DIRECTION
                0... .... .... .... = Pen provided: No
                .000 0000 0011 1101 = Type: DIRECTION (61)
                Length: 1
            Field (3/4): dataLinkFrameSize
                0... .... .... .... = Pen provided: No
                .000 0001 0011 1000 = Type: dataLinkFrameSize (312)
                Length: 2
            Field (4/4): dataLinkFrameSection
                0... .... .... .... = Pen provided: No
                .000 0001 0011 1011 = Type: dataLinkFrameSection (315)
                Length: 65535 [i.e.: "Variable Length"]

Frame 7: 157 bytes on wire (1256 bits), 157 bytes captured (1256 bits) on interface pkt2, id 0
    Interface id: 0 (pkt2)
        Interface name: pkt2
    Encapsulation type: Ethernet (1)
    Arrival Time: Sep 19, 2022 16:41:49.073824363 UTC
    [Time shift for this packet: 0.000000000 seconds]
    Epoch Time: 1663605709.073824363 seconds
    [Time delta from previous captured frame: 3.091416682 seconds]
    [Time delta from previous displayed frame: 3.091416682 seconds]
    [Time since reference or first frame: 35.188356863 seconds]
    Frame Number: 7
    Frame Length: 157 bytes (1256 bits)
    Capture Length: 157 bytes (1256 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:udp:cflow]
Ethernet II, Src: 02:aa:01:10:01:01 (02:aa:01:10:01:01), Dst: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
    Destination: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
        Address: 9a:51:3a:41:5c:73 (9a:51:3a:41:5c:73)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 02:aa:01:10:01:01 (02:aa:01:10:01:01)
        Address: 02:aa:01:10:01:01 (02:aa:01:10:01:01)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 20.1.1.2, Dst: 20.1.1.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 143
    Identification: 0x34a4 (13476)
    Flags: 0x00
        0... .... = Reserved bit: Not set
        .0.. .... = Don't fragment: Not set
        ..0. .... = More fragments: Not set
    Fragment Offset: 0
    Time to Live: 250
    Protocol: UDP (17)
    Header Checksum: 0x61b5 [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 20.1.1.2
    Destination Address: 20.1.1.1
User Datagram Protocol, Src Port: 50151, Dst Port: 1000
    Source Port: 50151
    Destination Port: 1000
    Length: 123
    [Checksum: [missing]]
    [Checksum Status: Not present]
    [Stream index: 0]
    [Timestamps]
        [Time since first frame: 35.188356863 seconds]
        [Time since previous frame: 3.091416682 seconds]
    UDP payload (115 bytes)
Cisco NetFlow/IPFIX
    Version: 10
    Length: 115
    Timestamp: Sep 19, 2022 16:41:48.000000000 UTC
        ExportTime: 1663605708
    FlowSequence: 6
    Observation Domain Id: 16842752
    Set 1 [id=1024] (1 flows)
        FlowSet Id: (Data) (1024)
        FlowSet Length: 99
        [Template Frame: 2]
        Flow 1
            Enterprise Private entry: (Juniper Networks, Inc.) Type 137: Value (hex bytes): 18 00 01 44
            Direction: Ingress (0)
            Data Link Frame Size: 87
            Data Link Frame Section: 02aa01100100b644665e337c080045000049000000000011a4a00a0101020a01010103e8…
                String_len_short: 87
```

## Inline monitoring decoder
you can find an inline-monitoring decoder [here](https://github.com/ARD92/imon-decoder) to decode flows which are encapsulated within GTP headers.

