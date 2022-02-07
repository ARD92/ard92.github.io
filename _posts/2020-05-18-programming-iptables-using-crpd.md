---
layout: post
title: Programming IP table rules using cRPD
author: Aravind
tags: junos crpd juniper
---
Sometimes you may need a feature which is not natively available in junos. Given the timeline how to solve such problems ? Juniper has a great framework withing mgd which supports creating custom yang models and tie an action script underneath to bring in new RPC calls and configuration hierarchy. cRPD having the ability to run on linux provides some nice capability to manage the underlying linux apps to be controlled using Junos interface and expose the same over netconf. By doing so, we could use any controller(ODL for example) to make rpc calls to cRPD which under the hood would take the necessary actions as defined in the action script and the config script configures as per the yang model. Given this capability, we can try to program Iptables using cRPD using yang and action script as an example and use this a stop gap until the fetaure is available natively.  

To access the code, clone the directory from [here](https://github.com/ARD92/yang/tree/master/yang/iptable_program)

cRPD should have iptables installed. You can do so by running 

``` 
apt install -y iptables 
```
validate if iptables is installed.
```
iptables --version
```

## Steps to load the yang model and necessary scripts 

1. copy the files onto cRPD under a specific directory
``` 
docker cp <folder> <crpd>:/home 
```

2. enter cli mode by typing "cli" if under shell 

3. use the command to load the yang packages. While loading, it will validate the yang files and will indicate if any errors are present in the file.
```
request system yang add package iptables module [ firewall.yang junos-extension.yang junos-common-odl-extensions.yang ] action-script firewall_action_script.py
```

4. once the above is load , navigate to the directory from shell and start the config script. This would run as a daemon.
```
python firewall_config_script.py start
```

5. Configure the below so the loaded yang model can work
```
# set system commit xpath
# set system commit constraints direct-access
# set system commit notification configuration-diff-format xml
# set system scripts language python
```

## Test programming the IPtables 
```
set firewall:firewall policy 5_TUPLE from sourceIp 1.1.1.1
set firewall:firewall policy 5_TUPLE from destIp 2.2.2.2
set firewall:firewall policy 5_TUPLE from protocol TCP
set firewall:firewall policy 5_TUPLE from sourcePort 8000
set firewall:firewall policy 5_TUPLE from destPort 9000
set firewall:firewall policy 5_TUPLE from direction INPUT
set firewall:firewall policy 5_TUPLE then action ACCEPT
set firewall:firewall policy 5_TUPLE then predefined-table predefined-chain filter
```
OUTPUT

```
root@9174dce90317> show firewall table filter chain
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -s 1.1.1.1 -p TCP -d 2.2.2.2 --sport 8000 --dport 9000 -j ACCEPT  -t filter 
```

The above can also be configured over netconf by using respective xml structure. The usual rpc calls such as <edit-config> and <get> calls would also work.

## Additional test cases 
```
set firewall:firewall policy 5_TUPLE from sourceIp 1.1.1.1
set firewall:firewall policy 5_TUPLE from destIp 2.2.2.2
set firewall:firewall policy 5_TUPLE from protocol TCP
set firewall:firewall policy 5_TUPLE from sourcePort 8000
set firewall:firewall policy 5_TUPLE from destPort 9000
set firewall:firewall policy 5_TUPLE from direction INPUT
set firewall:firewall policy 5_TUPLE then action ACCEPT
set firewall:firewall policy 5_TUPLE then predefined-table predefined-chain filter

set firewall:firewall policy ICMP_REJECT from protocol ICMP
set firewall:firewall policy ICMP_REJECT from icmp-type echo-request
set firewall:firewall policy ICMP_REJECT from direction OUTPUT
set firewall:firewall policy ICMP_REJECT then action REJECT
set firewall:firewall policy ICMP_REJECT then predefined-table predefined-chain filter

set firewall:firewall policy MATCH_DSCP from sourceIp 10.10.1.1
set firewall:firewall policy MATCH_DSCP from DSCP set-dscp 20
set firewall:firewall policy MATCH_DSCP from direction PREROUTING
set firewall:firewall policy MATCH_DSCP then action ACCEPT
set firewall:firewall policy MATCH_DSCP then predefined-table predefined-chain mangle

set firewall:firewall policy SET_DSCP from sourceIp 20.20.1.2/24
set firewall:firewall policy SET_DSCP from DSCP set-dscp-class be
set firewall:firewall policy SET_DSCP from direction PREROUTING
set firewall:firewall policy SET_DSCP then action DSCP
set firewall:firewall policy SET_DSCP then predefined-table predefined-chain mangle

set firewall:firewall policy SET_CLASSIFY from sourceIp 30.1.1.1
set firewall:firewall policy SET_CLASSIFY from protocol UDP
set firewall:firewall policy SET_CLASSIFY from sourcePort 4000
set firewall:firewall policy SET_CLASSIFY from set-class 200:300
set firewall:firewall policy SET_CLASSIFY from direction POSTROUTING
set firewall:firewall policy SET_CLASSIFY then action CLASSIFY
set firewall:firewall policy SET_CLASSIFY then predefined-table predefined-chain mangle

set firewall:firewall policy TTL from sourceIp 40.1.10.1
set firewall:firewall policy TTL from ttl 30
set firewall:firewall policy TTL from direction INPUT
set firewall:firewall policy TTL then action DROP
set firewall:firewall policy TTL then predefined-table predefined-chain filter

set firewall:firewall policy MATCH_MSS from protocol TCP
set firewall:firewall policy MATCH_MSS from direction INPUT
set firewall:firewall policy MATCH_MSS from max-seg-size 1800
set firewall:firewall policy MATCH_MSS then action DROP


set firewall:firewall policy MATCH_TOS from sourceIp 20.10.20.10/24
set firewall:firewall policy MATCH_TOS from protocol TCP
set firewall:firewall policy MATCH_TOS from sourcePort 840
set firewall:firewall policy MATCH_TOS from set-tos 20
set firewall:firewall policy MATCH_TOS from set-class 10:20
set firewall:firewall policy MATCH_TOS from direction POSTROUTING
set firewall:firewall policy MATCH_TOS then action CLASSIFY
set firewall:firewall policy MATCH_TOS then predefined-table predefined-chain mangle


set firewall:firewall policy SET_TOS from protocol UDP
set firewall:firewall policy SET_TOS from sourcePort 1000:2000
set firewall:firewall policy SET_TOS from set-tos 20
set firewall:firewall policy SET_TOS from direction POSTROUTING
set firewall:firewall policy SET_TOS then action TOS
set firewall:firewall policy SET_TOS then predefined-table predefined-chain mangle

set firewall:firewall policy SET_MARK from mac 00:01:02:03:04:05
set firewall:firewall policy SET_MARK from packetMark 50
set firewall:firewall policy SET_MARK from direction INPUT
set firewall:firewall policy SET_MARK then action MARK
set firewall:firewall policy SET_MARK then predefined-table predefined-chain filter

set firewall:firewall policy FRAG_LENGTH from protocol TCP
set firewall:firewall policy FRAG_LENGTH from sourcePort 1000
set firewall:firewall policy FRAG_LENGTH from sourcePort 800
set firewall:firewall policy FRAG_LENGTH from sourcePort 900
set firewall:firewall policy FRAG_LENGTH from fragment
set firewall:firewall policy FRAG_LENGTH from direction OUTPUT
set firewall:firewall policy FRAG_LENGTH from length 1400:1500
set firewall:firewall policy FRAG_LENGTH then action DROP
set firewall:firewall policy FRAG_LENGTH then predefined-table predefined-chain raw

set firewall:firewall policy DNAT from destIp 192.168.1.10/24
set firewall:firewall policy DNAT from direction OUTPUT
set firewall:firewall policy DNAT from to-destination 50.1.50.1-50.1.50.10
set firewall:firewall policy DNAT then action DNAT
set firewall:firewall policy DNAT then predefined-table predefined-chain nat

set firewall:firewall policy SNAT from sourceIp 10.1.10.1/24
set firewall:firewall policy SNAT from direction POSTROUTING
set firewall:firewall policy SNAT from to-source 192.168.10.1-192.168.10.10
set firewall:firewall policy SNAT then action SNAT
set firewall:firewall policy SNAT then predefined-table predefined-chain nat

set firewall:firewall policy RATE_LIMIT from sourceIp 30.30.30.0/24
set firewall:firewall policy RATE_LIMIT from direction INPUT
set firewall:firewall policy RATE_LIMIT from rate-limit-packets limit-packets 100/second
set firewall:firewall policy RATE_LIMIT from rate-limit-packets limit-burst 5
set firewall:firewall policy RATE_LIMIT then action DROP
set firewall:firewall policy RATE_LIMIT then predefined-table predefined-chain filter

set firewall:firewall policy MATCH_TCP_FLAG from sourceIp 192.168.10.10
set firewall:firewall policy MATCH_TCP_FLAG from protocol TCP
set firewall:firewall policy MATCH_TCP_FLAG from tcp-flags FIN
set firewall:firewall policy MATCH_TCP_FLAG from tcp-flags SYN
set firewall:firewall policy MATCH_TCP_FLAG from direction INPUT
set firewall:firewall policy MATCH_TCP_FLAG then action DROP

set firewall:firewall policy SET_MaxSegSize from sourceIp 50.1.1.1/20
set firewall:firewall policy SET_MaxSegSize from protocol TCP
set firewall:firewall policy SET_MaxSegSize from tcp-flags SYN
set firewall:firewall policy SET_MaxSegSize from direction INPUT
set firewall:firewall policy SET_MaxSegSize from max-seg-size 1800
set firewall:firewall policy SET_MaxSegSize then action TCPMSS

set firewall:firewall policy MATCH_MARK from protocol TCP
set firewall:firewall policy MATCH_MARK from packetMark 200
set firewall:firewall policy MATCH_MARK from direction OUTPUT
set firewall:firewall policy MATCH_MARK from to-ports 8000-9000
set firewall:firewall policy MATCH_MARK then action REDIRECT
set firewall:firewall policy MATCH_MARK then predefined-table predefined-chain nat

set firewall:firewall policy CONNSTATE from connState new
set firewall:firewall policy CONNSTATE from connState Established
set firewall:firewall policy CONNSTATE from direction INPUT
set firewall:firewall policy CONNSTATE then action ACCEPT


set firewall:firewall policy CONN_LIMIT from sourceIp 20.1.2.2
set firewall:firewall policy CONN_LIMIT from connlimit 20
set firewall:firewall policy CONN_LIMIT from direction INPUT
set firewall:firewall policy CONN_LIMIT then action DROP
set firewall:firewall policy CONN_LIMIT then predefined-table predefined-chain filter
```
All the above can be installed at once. Once installed, the references against the policy name is maintained in a json file for easy handling to delete if needed.

```
{
    "5_TUPLE": "iptables -A INPUT -s 1.1.1.1 -p TCP -d 2.2.2.2 --sport 8000 --dport 9000 -j ACCEPT  -t filter ",
    "DNAT": "iptables -A OUTPUT -d 192.168.1.10/24 -j DNAT  -t nat --to-destination 50.1.50.1-50.1.50.10 ",
    "FRAG_LENGTH": "iptables -A OUTPUT -p TCP -m multiport --source-port 1000,800,900 -m length --length 1400:1500 -j DROP  -t raw -f ",
    "ICMP_REJECT": "iptables -A OUTPUT -p ICMP -j REJECT  -t filter --icmp-type echo-request ",
    "MATCH_DSCP": "iptables -A PREROUTING -s 10.10.1.1 -j ACCEPT  -t mangle -m dscp --dscp 20 ",
    "MATCH_MARK": "iptables -A OUTPUT -p TCP -j DROP  -t nat -m mark --mark 200 ",
    "MATCH_MSS": "iptables -A INPUT -p TCP -j DROP -m tcpmss --mss 1800 ",
    "MATCH_TCP_FLAG": "iptables -A INPUT -s 192.168.10.10 -p TCP -j DROP --tcp-flags ALL FIN,SYN ",
    "MATCH_TOS": "iptables -A POSTROUTING -s 20.10.20.10/24 -p TCP --sport 840 -j CLASSIFY  -t mangle --set-class 10:20 -m tos --tos 20 ",
    "RATE_LIMIT": "iptables -A INPUT -s 30.30.30.0/24 -m limit --limit 100/second --limit-burst 5 -j DROP  -t filter ",
    "SET_CLASSIFY": "iptables -A POSTROUTING -s 30.1.1.1 -p UDP --sport 4000 -j CLASSIFY  -t mangle --set-class 200:300 ",
    "SET_DSCP": "iptables -A PREROUTING -s 20.20.1.2/24 -j DSCP  -t mangle --set-dscp-class be ",
    "SET_MARK": "iptables -A INPUT -j MARK  -t filter -m mac --mac-source 00:01:02:03:04:05 --set-mark 50 ",
    "SET_MaxSegSize": "iptables -A INPUT -s 50.1.1.1/20 -p TCP -j TCPMSS --tcp-flags ALL SYN --set-mss 1800 ",
    "SET_TOS": "iptables -A POSTROUTING -p UDP --sport 1000:2000 -j TOS  -t mangle --set-tos 20 ",
    "SNAT": "iptables -A POSTROUTING -s 10.1.10.1/24 -j SNAT  -t nat --to-source 192.168.10.1-192.168.10.10 ",
    "TTL": "iptables -A INPUT -s 40.1.10.1 -j DROP  -t filter -m ttl  --ttl 30 "
}
```

## more about commit script and how it works 
* https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-script-automation-commit-script-works.html
* https://www.juniper.net/documentation/en_US/junos/topics/task/program/netconf-yang-scripts-action-creating.html
* https://www.juniper.net/documentation/en_US/junos/topics/task/program/netconf-yang-rpcs-custom-creating.html
