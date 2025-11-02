---
layout: post
title:  Topology generation for s2 observability simulator
tags: python ai_ml
---

When using the S2 observability simulator, there are a few files that needs to be written in order for it to work. The files have to follow a certain schema in order for things to seamlessly ingested into the selector platform. This contains the prompts used in gemini-cli to help generate the files 

Gemini-cli allows to write instructions that has to be followed. The context is loaded in hierarchy, first looking through `~/.gemini/GEMINI.md` file and then through respective project folder `<path to project>/.gemini/GEMINI.md`  

Use the below configuration in order to help generate the files and save it under `~/.gemini/GEMINI.md` 

```
# GEMINI.md
## General Instructions
- you are a networking expert with CCIE level knowledge

## Topology Creation Instructions
- when asked for topology creation,always create csv with headers a_node,z_node,a_intf,z_intf,a_traffic_profile,z_traffic_profile,namespace
- traffic_profiles always follows formats <value>-1-1. Value ranges from 10 to 20
- intf always follows format xe-0/0/<value> or et-0/0/<value>
- device naming convention follows <2 character state name><PE or SW or FW><CIS or JNP or NOK><value>
- ensure the topology has entries of devices connected in reverse direction
- if a namespace is not provided, use mcp and save file as topology.csv
- generate a unique list of the devices and save as device_list.csv
- for the list of devices mark the following columns device, site = first 2 characters, type = character 2 - 4, vendor = character 4-7, lat = latitue of site , lng = longitude of site, namespace = mcp

## BGP Configuration
- when asked for bgp configuration , create csv with headers type, device, device_ip, remote_device, remote_ip, local_as, remote_as, interface, provider
- device_ip and remote_ip are the bgp ipv4 addresses
- type is either E for external or I for internal bgp neighborship
- local as and remote as should follow bgp configuration
- provider can be internal for I bgp connections and any ISP for an external connection
- interface would be interface overwhich bgp is configured

## Device Configuration
- for every generated topology, generate the router or switch configuration
- create a directory configs and store config based on the device name and .txt extensions
- config should contain protocols, interfaces , route policies
- bgp configuration should match with what has been defined in the csv file
```

## Start Gemini and use the following prompt 

* generate a network topology with 2 data centers connecting each other.  Each of the data center contains a 3 stage CLOS architecture. Save the file in the current working directory
* update configuration to contain bgp underlay and evpn overlay configuration
* generate configurations for 2 devices where bgp neighborship is deleted and store them in format devicename_anomaly.txt

### Outputs
```
(mcp-topo-gen) aravindprabhakar@Aravinds-MBP mcp-topo-gen % ls -l
total 80
-rw-r--r--@  1 aravindprabhakar  staff  5811 Nov  2 09:45 bgp_config.csv
drwxr-xr-x@ 38 aravindprabhakar  staff  1216 Nov  2 09:53 configs
-rw-r--r--@  1 aravindprabhakar  staff  1615 Nov  2 09:45 device_list.csv
-rw-r--r--@  1 aravindprabhakar  staff  3967 Nov  2 09:45 topology.csv
```

## Use this in S2 observability tool

* Step1: Upload the device_list.csv into selector >> metastore2 >> device_inventory 
![device_list](/images/device_inventory.png){:class="img-responsive"}

* Step2: Upload topology using 
    ```
    python3 generate_all_stats.py -t topology.csv -i device_list.csv -a replay -f topology.csv
    ```

* Step3: Ensure all files under the config directory are copied under configs dir under the s2-observability-simulator/configs

* Step4: Start steady state traffic 
    ```
    python generate_all_stats.py -t topology.csv -a steadystate -b bgp_config.csv
    ```

* Step5: Validate dashboard
![dashboard](/images/dashboard.png){:class="img-responsive"}
