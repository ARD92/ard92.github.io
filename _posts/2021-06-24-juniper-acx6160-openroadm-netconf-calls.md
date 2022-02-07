---
layout: post
title: Netconf calls for Openroadm devices  
author: Aravind
tags: juniper junos
---

Junipers ACX6160-T is a pure L1 transponder supporting multi 100G connections over CFP2DCO and QSFP28s. The configuration follows openroadm model and the expectation is the openroadm controller would be programming the transponder over Netconf 

```
root@re0> show chassis hardware
Hardware inventory:
Item             Version  Part number  Serial number     Description
Chassis                                EQ3xxxxxxxx      ACX6160-T
PSM 0            REV 02   740-053639   1GD38260759       JPSU-850W-DC-AFO
PSM 1            REV 02   740-053639   1GD38260763       JPSU-850W-DC-AFO
Routing Engine 0 REV 01   650-090154   EQ3118AT0023      ACX6160-T
FPC 0                     BUILTIN      BUILTIN           ACX6160-T
  PIC 0                   BUILTIN      BUILTIN           8X100G-QSFP28
    Xcvr 2       0        NON-JNPR     T17C33056         QSFP-100GBASE-LR4
    Xcvr 4       0        NON-JNPR     T17C33052         QSFP-100GBASE-LR4
  PIC 1                   BUILTIN      BUILTIN           4X200G-CFP2DCO
    Xcvr 0       REV 01   740-087314   1TCBY331007       CFP2 DCO
    Xcvr 2       REV 01   740-087314   1TCBY331001       CFP2 DCO
Fan Tray 0                                               ACX6160-T Fan Tray
Fan Tray 1                                               ACX6160-T Fan Tray
Fan Tray 2                                               ACX6160-T Fan Tray
Fan Tray 3                                               ACX6160-T Fan Tray
Fan Tray 4                                               ACX6160-T Fan Tray
```

Below are some of the Netconf calls for reference 

# References 
- https://www.google.com/search?q=juniper+acx6160&oq=juniper+acx6160&aqs=chrome..69i57j69i64.3258j0j1&sourceid=chrome&ie=UTF-8

# Netconf RPCs

## Restart RPC
Cold and warm restarts are supported 
```
<rpc message-id="m-91" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<restart xmlns="http://org/openroadm/de/operations">
<option>warm</option>
</restart>
</rpc>

 
<rpc message-id="m-91" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<restart xmlns="http://org/openroadm/de/operations">
<option>cold</option>
</restart>
</rpc>
```

## Create Tech info
Since the ACX6160-T is a single shelf box, only shelf-0 is supported 
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <create-tech-info xmlns="http://org/openroadm/device">
    <shelf-id>shelf-0</shelf-id>
    <log-option>all</log-option>
  </create-tech-info>
</rpc
```

## Show alarms
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter>
      <active-alarm-list xmlns="http://org/openroadm/alarm"/>
    </filter>
  </get>
</rpc>
```

## info container
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter>
      <org-openroadm-device xmlns="http://org/openroadm/device"/><info/>
    </filter>
  </get>
</rpc>
```

## Get current PM
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter>
      <current-pm-list xmlns="org-openroadm-pm"/>
    </filter>
  </get>
</rpc>
```

## Get historical PM
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <collect-historical-pm-file xmlns="http://org/openroadm/pm">
<from-bin-number>1</from-bin-number>
<to-bin-number>1</to-bin-number>
<granularity>15min</granularity>
  </collect-historical-pm-file>
</rpc>
```

## DB init the node
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<database-init xmlns="org-openroadm-database"/>
</rpc>
```
## Cancel validation timer
```
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<cancel-validation-timer xmlns=org-openroadm-swdl>
 <accept>true</accept>
</cancel-validation-timer>
</rpc>
```

## Led control
```
<rpc>
<led-control xmlns="org-openroadm-device">
<shelf-name>shelf-0</shelf-name>
<enabled>false</enabled>
</led-control>
</rpc>
```

## Upgrade firmware
```
<rpc>
<fw-update xmlns="http://org/openroadm/fwdl">
<circuit-pack-name>xcvr-0/0/0</circuit-pack-name>
</fw-update>
<rpc>
```

## ISSU upgrade procedure
make sure to change the file name accordingly!
```
<rpc message-id="m-20" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<sw-stage xmlns="http://org/openroadm/de/swdl">
<filename>loadApril16-t.iso</filename>
</sw-stage>
</rpc>
 
<rpc message-id="m-20" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<sw-activate xmlns="http://org/openroadm/de/swdl">
<version>-19.2I20190415083016-EVO_rashid</version>
<validationTimer>00-60-00</validationTimer>
</sw-activate>
</rpc>
 
<rpc message-id="m-20" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<cancel-validation-timer xmlns="http://org/openroadm/de/swdl">
<accept>true</accept>
</cancel-validation-timer>
</rpc>
```

## DB backup
```
<rpc>
<db-backup xmlns="http://org/openroadm/database">
<filename> test_backup.dbs </filename>
</db-backup>
</rpc>
```

## File transfer
```
<rpc>
<transfer xmlns="http://org/openroadm/file-transfer">
<action>upload</action>
<local-file-path>test_backup.dbs</local-file-path>
<remote-file-path>sftp://openroadm:openroadm@10.228.66.9/var/openroadm/test_backup.dbs</remote-file-path>
</transfer>
</rpc>
```

## Get-config examples
```
<rpc>
<get-config>
<source>
<candidate/>
</source>
<filter type="subtree">
<configuration>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
    <name>re0:mgmt-0</name>
</interface>
</org-openroadm-device>
</configuration>
</filter>
</get-config>
</rpc>


<rpc>
<get-config>
<source>
<candidate/>
</source>
<filter type="subtree">
<configuration>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
</interface>
</org-openroadm-device>
</configuration>
</filter>
</get-config>
</rpc>


<rpc>
<get-config>
<source>
<running/>
</source>
<filter type="subtree">
<configuration>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
    <name>ett-0/0/2</name>
</interface>
</org-openroadm-device>
</configuration>
</filter>
</get-config>
</rpc>
```

## Enable test signal
```
<rpc message-id="m-38" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
<name>ord_if_odu_line_0</name>
<administrative-state>maintenance</administrative-state>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<maint-testsignal>
<enabled>true</enabled>
<type>term</type>
<testPattern>PRBS</testPattern>
</maint-testsignal>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>

 
<rpc message-id="m-25" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<commit/>
</rpc>
```

## Disable test signal
```
<rpc message-id="m-43" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
<name>ord_if_odu_line_0</name>
<administrative-state>inService</administrative-state>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<maint-testsignal>
<enabled>false</enabled>
</maint-testsignal>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>

<rpc message-id="m-25" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<commit/>
</rpc>
```

## Loopback enable
```
<rpc> 
    <edit-config> 
        <target> 
            <candidate/> 
        </target> 
        <config> 
            <configuration> 
                <!-- tag elements representing the data to incorporate -->
                <org-openroadm-device xmlns="http://org/openroadm/device">
                        <interface>
                            <name>ord_if_otu_line_0</name>
                            <otu xmlns="http://org/openroadm/otn-otu-interfaces">
                                <maint-loopback>
                                    <enabled>true</enabled>
                                    <type>term</type>
                                </maint-loopback>
                            </otu>
                        </interface>
                </org-openroadm-device>             
            </configuration> 
        </config> 
    </edit-config> 
</rpc> 


<rpc message-id="m-25" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<commit/>
</rpc>
```

## Disable loopback
```
<rpc> 
    <edit-config> 
        <target> 
            <candidate/> 
        </target> 
        <config> 
            <configuration> 
                <!-- tag elements representing the data to incorporate -->
                <org-openroadm-device xmlns="http://org/openroadm/device">
                    <interface>
                        <name>ord_if_otu_line_0</name>
                        <otu xmlns="http://org/openroadm/otn-otu-interfaces">
                            <maint-loopback>
                                <enabled>false</enabled>
                            </maint-loopback>
                        </otu>
                    </interface>
                </org-openroadm-device>
            </configuration> 
        </config> 
    </edit-config> 
</rpc>


 
<rpc message-id="m-25" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<commit/>
</rpc>
```

## Configure mgmt interface
```
<rpc>
<edit-config>
<target>
<candidate/>
</target>
<config>
    <org-openroadm-device xmlns="http://org/openroadm/device">
            <circuit-packs>
                <circuit-pack-name>scm-0</circuit-pack-name>
                <administrative-state>inService</administrative-state>
                <equipment-state>reserved-for-facility-available</equipment-state>
                <circuit-pack-mode>NORMAL</circuit-pack-mode>
                <shelf>shelf-0</shelf>
                <slot>slot-8</slot>
                <subSlot>slot-0</subSlot>
                <due-date>2018-12-31T00:00:00Z</due-date>
                <ports>
                    <port-name>port-0</port-name>
                    <port-type>ethernet-port</port-type>
                    <port-qual>roadm-internal</port-qual>
                    <administrative-state>inService</administrative-state>
                </ports>
                <circuit-pack-type>SCM</circuit-pack-type>
            </circuit-packs>
    </org-openroadm-device>
</config>
</edit-config>
</rpc>

<rpc>
<edit-config>
<target>
<candidate/>
</target>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
<name>re0:mgmt-0</name>
<description>management-interface</description>
<type xmlns:typeethernetCsmacd="http://org/openroadm/interfaces">typeethernetCsmacd:ethernetCsmacd</type>
<administrative-state>inService</administrative-state>
<supporting-circuit-pack-name>scm-0</supporting-circuit-pack-name>
<supporting-port>port-0</supporting-port>
<ethernet xmlns="http://org/openroadm/ethernet-interfaces">
<auto-negotiation>enabled</auto-negotiation>
</ethernet>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>


<rpc><commit/></rpc>

======== Get call to verify =========
<rpc>
<get>
<filter>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface>
<name>re0:mgmt-0</name>
</interface>
</org-openroadm-device>
</filter>
</get>
</rpc>
```

## Provision 100GE circuit

### Topology
```
+----------+         +-------------+               +-------------+            +-----------+
|          |  100GE  |eth0/0/0     +---------------+och-0/1/0    |    100GE   |           |
|  tester1 +---------+    och-0/1/0|    otu4       |     eth-0/0/0------------+  tester2  |
+----------+         +-------------+               +--------------            +-----------+
```

### Netconf calls 
```
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_eth_client_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/0/0</supporting-circuit-pack-name>
<administrative-state>inService</administrative-state>
<description>ett-interface-client_0</description>
<ethernet xmlns="http://org/openroadm/ethernet-interfaces">
<speed>100000</speed>
</ethernet>
<type xmlns:x="http://org/openroadm/interfaces">x:ethernetCsmacd</type>
<supporting-port>port-0/0/0</supporting-port>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_och_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/1/0</supporting-circuit-pack-name>
<administrative-state>inService</administrative-state>
<description>och-interface-line</description>
<supporting-port>port-0/1/0</supporting-port>
<type xmlns:x="http://org/openroadm/interfaces">x:opticalChannel</type>
<och xmlns="http://org/openroadm/optical-channel-interfaces">
<rate xmlns:x="http://org/openroadm/common-types">x:R100G</rate>
<frequency>193.1</frequency>
<modulation-format>qpsk</modulation-format>
<transmit-power>-3.00</transmit-power>
</och>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_otu_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_och_line_0</supporting-interface>
<administrative-state>inService</administrative-state>
<description>otu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOtu</type>
<otu xmlns="http://org/openroadm/otn-otu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:OTU4</rate>
<fec>scfec</fec>
</otu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_odu_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_otu_line_0</supporting-interface>
<administrative-state>inService</administrative-state>
<description>odu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOdu</type>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:ODU4</rate>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
```

## Provision OTU4 circuit

### Topology
```
+----------+         +-------------+               +-------------+            +-----------+
|          |  otu4   |ord-0/0/0    +---------------+och-0/1/0    |    otu4    |           |
|  tester1 +---------+    och-0/1/0|    otu4       |     ord-0/0/0------------+  tester2  |
+----------+         +-------------+               +--------------            +-----------+
```
### Netconf calls 
```
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_otu_client_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/0/0</supporting-circuit-pack-name>
<supporting-port>port-0/0/0</supporting-port>
<administrative-state>inService</administrative-state>
<description>otu-interface-client</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOtu</type>
<otu xmlns="http://org/openroadm/otn-otu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:OTU4</rate>
<fec>rsfec</fec>
</otu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_odu_client_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_otu_client_0</supporting-interface>
<administrative-state>inService</administrative-state>
<description>odu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOdu</type>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:ODU4</rate>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_och_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/1/0</supporting-circuit-pack-name>
<administrative-state>inService</administrative-state>
<description>och-interface-line</description>
<supporting-port>port-0/1/0</supporting-port>
<type xmlns:x="http://org/openroadm/interfaces">x:opticalChannel</type>
<och xmlns="http://org/openroadm/optical-channel-interfaces">
<rate xmlns:x="http://org/openroadm/common-types">x:R100G</rate>
<frequency>193.1</frequency>
<modulation-format>qpsk</modulation-format>
<transmit-power>-3.00</transmit-power>
</och>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_otu_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_och_line_0</supporting-interface>
<administrative-state>inService</administrative-state>
<description>otu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOtu</type>
<otu xmlns="http://org/openroadm/otn-otu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:OTU4</rate>
<fec>scfec</fec>
</otu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_odu_line_0</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_otu_line_0</supporting-interface>
<administrative-state>inService</administrative-state>
<description>odu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOdu</type>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:ODU4</rate>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
 
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_otu_client_4</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/0/4</supporting-circuit-pack-name>
<supporting-port>port-0/0/4</supporting-port>
<administrative-state>inService</administrative-state>
<description>otu-interface-client</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOtu</type>
<otu xmlns="http://org/openroadm/otn-otu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:OTU4</rate>
<fec>rsfec</fec>
</otu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_odu_client_4</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_otu_client_4</supporting-interface>
<administrative-state>inService</administrative-state>
<description>odu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOdu</type>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:ODU4</rate>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_och_line_2</name>
<circuit-id>1234567890</circuit-id>
<supporting-circuit-pack-name>xcvr-0/1/2</supporting-circuit-pack-name>
<administrative-state>inService</administrative-state>
<description>och-interface-line</description>
<supporting-port>port-0/1/2</supporting-port>
<type xmlns:x="http://org/openroadm/interfaces">x:opticalChannel</type>
<och xmlns="http://org/openroadm/optical-channel-interfaces">
<rate xmlns:x="http://org/openroadm/common-types">x:R100G</rate>
<frequency>193.1</frequency>
<modulation-format>qpsk</modulation-format>
<transmit-power>-3.00</transmit-power>
</och>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_otu_line_2</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_och_line_2</supporting-interface>
<administrative-state>inService</administrative-state>
<description>otu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOtu</type>
<otu xmlns="http://org/openroadm/otn-otu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:OTU4</rate>
<fec>scfec</fec>
</otu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
 
<rpc message-id="m-45" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config>
<target>
<candidate/>
</target>
<default-operation>none</default-operation>
<config>
<org-openroadm-device xmlns="http://org/openroadm/device">
<interface xmlns:a="urn:ietf:params:xml:ns:netconf:base:1.0" a:operation="replace">
<name>ord_if_odu_line_2</name>
<circuit-id>1234567890</circuit-id>
<supporting-interface>ord_if_otu_line_2</supporting-interface>
<administrative-state>inService</administrative-state>
<description>odu-interface-line</description>
<type xmlns:x="http://org/openroadm/interfaces">x:otnOdu</type>
<odu xmlns="http://org/openroadm/otn-odu-interfaces">
<rate xmlns:x="http://org/openroadm/otn-common-types">x:ODU4</rate>
</odu>
</interface>
</org-openroadm-device>
</config>
</edit-config>
</rpc>
```
