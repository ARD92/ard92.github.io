---
layout: post
title: vlan id and native vlan ids
tags: Junos switching
---
# Topology considered

```
+-----------------+                   +-----------------+                +-----------------+
|                 |                   |                 |                |                 |
|                 |0/0/0         0/0/0|                 | 0/0/1    0/0/1 |                 |
|    EX1          +-------------------+     EX2         +----------------+     CE1         |
|                 |                   |                 |                |                 |
|                 |                   |                 |                |                 |
+-----------------+                   +-----------------+                +-----------------+
 ```
 
##  TEST CASE 1 : Hosts on same vlan member are reachable 

EX1 and EX3 are hosts directly connected over to EX2 . Both the interfaces of EX2 (0/0/0 and 0/0/1 ) belong to Vlan member red.
EX1 would be now reachable to EX3 because both belong to same broadcast domain. The Ports are access ports.
 
```
root@EX1# show interfaces ge-0/0/0
unit 0 {
    family inet {
        address 192.168.1.1/30;
    }
}
 
 
root@EX2# show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
ge-0/0/1 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
 
root@EX2# show vlans
blue {
    vlan-id 20;
}
red {
    vlan-id 10;
}
 
root@EX3# show interfaces ge-0/0/1
unit 0 {
    family inet {
        address 192.168.1.2/30;
    }
}
```

### Ping EX1 to EX3

```
root@s14-3# run ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=11.096 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=12.214 ms
```
##  TEST CASE 2 :

When 0/0/1 and 0/0/0 of EX2 are configured as access ports with a single vlan member (red) the iRBs would be able to ping from EX1 to EX3

``` 
root@s14-3# run ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=22.128 ms
```

``` 
root@s14-3# show vlans
red {
    vlan-id 10;
    l3-interface irb.10;
}
 
{master:0}[edit]
root@EX1# show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
irb {
    unit 10 {
        family inet {
            address 192.168.1.1/30;
        }
    }
}
 
 
root@EX2 # show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
ge-0/0/1 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
 
{master:0}[edit]
root@EX2# show vlans
blue {
    vlan-id 20;
}
red {
    vlan-id 10;
}
 
root@EX3# show interfaces
ge-0/0/1 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
irb {
    unit 10 {
        family inet {
            address 192.168.1.2/30;
        }
    }
}
 
{master:0}[edit]
root@EX3# show vlans
red {
    vlan-id 10;
    l3-interface irb.10;
}
```
 
##  TEST CASE 3 : Make port 0/0/1 as trunk on EX2 

When the port is made as a trunk port on 0/0/1 on EX2. It is expected for EX3 interface 0/0/0 to be as a trunk interface.
Port 0/0/0 on EX2 is access interface with a vlan member. IRBs now ping each other .
 
If EX1 is behaving as a host and lands on 0/0/0 of EX2 into vlan member red. And if the 0/0/1 of EX2 is a trunk interface having red and blue. EX1 and EX3 would be reachable still.

``` 
root@EX1# run ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=11.245 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=11.159 ms  
```

``` 
root@EX1# show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
irb {
    unit 10 {
        family inet {
            address 192.168.1.1/30;
        }
    }
}
 
root@EX1# show vlans
red {
    vlan-id 10;
    l3-interface irb.10;
}
 
root@EX2# show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
ge-0/0/1 {
    unit 0 {
        family ethernet-switching {
            interface-mode trunk;
            vlan {
                members [ red blue ];
            }
        }
    }
}
 
root@EX2# show vlans
blue {
    vlan-id 20;
}
red {
    vlan-id 10;
}
 
 
 
root@EX3# show interfaces
ge-0/0/1 {
    unit 0 {
        family ethernet-switching {
            interface-mode trunk;
            vlan {
                members red;
            }
        }
    }
}
irb {
    unit 10 {
        family inet {
            address 192.168.1.2/30;
        }
    }
}
``` 
 
Note: If the EX3 interface 0/0/1 is not configured as trunk, then traffic would not pass as it has be trunk intf to pass traffic. Unless we allow native vlan tagging.
 
 
## TEST CASE 4 
EX1 behaving as a host having an IP 192.168.1.1/30 . EX3 is behaving as a host with IP 192.168.1.2/30.  The port 0/0/0 on EX2 is an access port belonging to vlan member “red”. The port 0/0/1 on EX2
Is a trunk interface with members “Red” and “blue”.  The EX3 behaving as a host is dipping into 0/0/1 trunk intf of EX2
Test ping from EX1 to EX3 . Ping should fail at this point. Because any untagged packets will be dropped by the trunk interface on EX2. In order to allow untagged packets, use the “native-vlan-tagging” knob and assign the respective vlan to it.
 
Assign native vlan id of 10 so any untagged packet as be received with vlan 10 which belongs to vlan member red consisting of interfaces .
 
Initial before configuring native vlan

```
root@EX1# show interfaces
ge-0/0/0 {
    unit 0 {
        family inet {
            address 192.168.1.1/30;
        }
    }
}
 
root@EX2# show interfaces
ge-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members red;
            }
        }
    }
}
ge-0/0/1 {
    inactive: native-vlan-id 10; <<<< have deactivated
    unit 0 {
        family ethernet-switching {
            interface-mode trunk;
            vlan {
                members [ red blue ];
            }
        }
    }
}

 
root@EX3# show interfaces ge-0/0/1
unit 0 {
    family inet {
        address 192.168.1.2/30;
    }
}
```

``` 
root@EX1# run ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
^C
--- 192.168.1.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

 
Activate the native vlan tag to accept untagged packets

``` 
root@EX2 # activate interfaces ge-0/0/1 native-vlan-id
 
root@EX1# run ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=21.816 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=11.148 ms
```


