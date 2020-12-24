---
layout: post
title: Programming IP table rules using cRPD
author: Aravind
---
# Programming IPtables using cRPD ? 
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


## more about commit script and how it works 
* https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-script-automation-commit-script-works.html
* https://www.juniper.net/documentation/en_US/junos/topics/task/program/netconf-yang-scripts-action-creating.html
* https://www.juniper.net/documentation/en_US/junos/topics/task/program/netconf-yang-rpcs-custom-creating.html
