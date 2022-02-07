---
layout: post
title: Daemonize python apps in cRPD
tags: junos linux crpd
---

# Daemonize python apps 
cRPD is Juniper's containerized routing stack which packs in a variety of features of capabilities that one can explore, starting with running it as an RR to running it as a PE router. While it packs in these features, there could be scenarios where one wants to bring in some additional capabilities (ex: 3rd party apps). How does one expose that to MGD ? Since cRPD packs in the MGD along, one can write custom yang modules to ensure cRPD can control these using Junos configs which can then be exposed over netconf in case an interface is needed. The underlying config script can be written to handle the custom yang modules allowing one to develop features rapidly when they aren't existing natively. 

Once the application is written one can also daemonize it and leverage the service functionality using runit which is a cross platform service supervision. Since runit is already packed in cRPD, it is very easy for one to leverage this and daemonize the application to ensure the app is restarted in scenarios such as system startup or the application is killed.

## Create the application

## Steps
1. ensure runit is already running. There are several processes in cRPD which leverages this and can be verified using the below 
    ```
    root@crpd03:/home/unicast-vxlan# ps aux
    USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root           1  0.0  0.0   4404   764 pts/0    Ss+  09:59   0:00 runit
    root          89  0.0  0.0   4412   732 ?        Ss   09:59   0:00 runsv rsyslog
    root          90  0.0  0.0   4412   828 ?        Ss   09:59   0:00 runsv license-check
    ...
    ...
    ```
2. Copy your python application to the location where you intend to run it from. You can mount the volume instead as well. In this case i am placing the file under /var/db/scripts/jet
    ```
    cp <file.py> /var/db/scripts/jet
    ```
3. Give executable permissions 
    ```
    chmod +x /var/db/scripts/jet/<file.py>
    ```
4. Create a package 
    ```
    Mkdir -p /etc/runit/<packagename>/
    ```
5. Create a run file in the above directory with contents to execute. The run file can be written accordingly based on requirements. Please refer to runit documentation [here](http://smarden.org/runit/index.html)
    ```
    #!/bin/sh -e
    Exec 2>&1
    Exec chpst -u root <path to script>
    ```
6. Provide executable permissions to the run file 
    ```
    Chmod +x /etc/runit/<packagename>/run
    ```
7. Create a sym link so that the service could be started 
    ```
    ln -s /etc/runit/<packagename>/ /etc/service/<packagename>
    ```

Once the above are completed, the service would now be started. The service file would be present under /etc/service/<packagename> which would inturn execute the python app defined inside the run file. 

### Managing the service 

- sv status <packagename>
- sv start <packagename>
- sv stop <packagename>

### Example 
I have created a service for a python app to create static unicast vxlan tunnels in cRPD and below are the respective captures. you can find the whole project [here](https://github.com/ARD92/yang/tree/master/yang/unicast-vxlan)

#### Check if app is running  
```
root@crpd01:/# ps aux | grep unicast-vxlan
root         812  0.0  0.0   4412  1192 ?        Ss   10:04   0:00 runsv unicast-vxlan
root         966  0.1  0.0  81100 23344 ?        S    14:39   0:00 python3 /var/db/scripts/jet/unicast-vxlan.py
root         970  0.0  0.0  11476  1004 pts/3    S+   14:40   0:00 grep --color=auto unicast-vxlan
```

#### Check status 
```
root@crpd01:/# sv status unicast-vxlan
run: unicast-vxlan: (pid 928) 12476s
```
#### Restart app 
```
root@crpd01:/# sv restart unicast-vxlan
ok: run: unicast-vxlan: (pid 966) 0s
```

#### Test by manually killing the app 
```
root@node1:/etc/runit# kill -9 2910
root@node1:/etc/runit# ps aux | grep unicast-vxlan
root        2885  0.0  0.0   4412   844 ?        Ss   09:17   0:00 runsv unicast-vxlan
root        2968  1.4  0.0  81108 23416 ?        S    09:20   0:00 python3 /var/db/scripts/jet/unicast-vxlan.py
root        2972  0.0  0.0  11476  1060 pts/9    S+   09:20   0:00 grep --color=auto unicast-vxlan
```
Notice the PID value changed from 2910 --> 2972. 

## References 
- http://smarden.org/runit/
- https://kchard.github.io/runit-quickstart/
