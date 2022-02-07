---
layout: post
title: Using Juniper Terraform Automation Framework (JTAF) to generate Junos terraform providers  
tags: junos juniper
---

# How to use JTAF  
As we use different IaC tools, terraform is one of the hot topics in the industry. It is by Hashicorp and is written in HCL. As part of Infrastructure as code, there are several overlaps with ansible but it offers other advantages over ansible and similar tools. In this we will see how to use JTAF to generate a Junos provider and program vSRX accordingly over netconf. 

more information on JTAF can be found [here](https://github.com/Juniper/junos-terraform)

## Clone the Yang repository 
```
git clone https://github.com/Juniper/yang.git
```
## clone the JTAF repository 

```
git clone https://github.com/Juniper/junos-terraform.git
```

## Install go and terraform 
In my case, i have the below go and terraform versions. you can download terraform and golang from the below 
* [terraform](https://www.terraform.io/downloads.html) 
* [golang](https://golang.org/dl/)

### Go version 
```
root@compute21:~# go version
go version go1.16.5 linux/amd64
```

### TF version 
```
root@compute21:~# terraform version
Terraform v1.0.2
on linux_amd64
```
## chose the yang file for which you need to generate a terraform provider 

Choose the yang file you want to generate providers on. In my example, i am choosing the securities conf file under junos-es directory. The exact file used is [here](https://github.com/Juniper/yang/blob/master/21.2/21.2R1/junos-es/conf/junos-es-conf-security%402019-01-01.yang)

Follow the procedure from JTAF readme to compile the files.

## Create the config.toml file 
create the below file as config.toml under the cmd/processYang directory.

* yangDir: Represents the directory where the yang files are present 
* providerDir: Represents the directory where providers would be created 
* xpathPath: This is not used when still processing Yang. It is used when using ProcessProvider. So any dummy value would also work. 
* fileType: can be text, xml or both. 

```
yangDir = "/home/aprabh/junos-terraform/vsrx"
providerDir = "/home/aprabh/junos-terraform/terraform_providers"
xpathPath = "/home/aprabh/junos-terraform/vsrx/xpath_test.xml"
fileType = "both"
```

### compile to generate the xpaths and xml files  
```
cd cmd/processYang
go build
./processYang -config config.toml
once you generate the yin, xml and xpaths, you can create the provider. 
```

### Example of how the generated xpath file looks like 
```
/security
/security/apply-groups
/security/apply-groups-except
/security/apply-macro
/security/apply-macro/name
/security/apply-macro/data
/security/apply-macro/data/name
/security/apply-macro/data/value
< ------- snipped --------- >

```
## Refer the xpaths for which resources need to be generated 
multiple xpaths can be refered. There would be multiple resources but a single binary. It is advisable to drill down as much as possible because certain keys like "name", "apply groups" are used in multiple hierarchies and it leads to confusion. The naming convention for those duplicated keywords would be with numbers . example name__1, name__2 etc.

### create sameple xpath file 
Here i am creating a sample xpath file under directory /home/aprabh/junos-terraform/vsrx/ as "xpath_test.xml"
The xpath i choose is to create zones for vSRX and place interfaces in the respective zones with system services and protocols to be enabled. 

```
<file-list>
        <xpath name="/security/zones/security-zone/interfaces"/>
</file-list>
```
The file-list tag can contain mutliple such xpaths for which we want to generate the provider. 

### Compile the code and process 
```
cd cmd/processProviders
go build
./processProviders -config config.toml
```

## Generate the provider
after the above, the files are APIs are generated in the providerDir mentioned in the config.toml file. 

compile the terraform provider binary using the below. The compilation depends on some of the files present under junos-terraform/terraform_providers/ so it is advisable to compile them there.

```
go build -o terraform-provider-junos-qfx
```

## Use the Generated provider 
The provider.go file shows the mapping of resource names to the function calls. For example in the above xpath generated provider "junos-qfx_SecurityZonesSecurity__ZoneInterfaces" is the resource name we need to refer to to create the zones and placing the interface in the created zone. The "junos-qfx_commit" is used to pass the config to the devices which helps in passing commit operation. 

```
        ResourcesMap: map[string]*schema.Resource{
            "junos-qfx_SecurityZonesSecurity__ZoneInterfaces": junosSecurityZonesSecurity__ZoneInterfaces(),
            "junos-qfx_commit": junosQFXCommit(),
        },
```

### Example of configuring vSRX with a new Zone 
```
terraform {
  required_providers {
    junos-qfx = {
      source = "10.85.47.161/ns/junos-qfx"
      version = "1.0"
    }
  }
}

provider "junos-qfx" {
    host = "10.102.144.2"
    port = 830
    username = "root"
    password = "Juniper123"
    sshkey = ""
}


resource "junos-qfx_SecurityZonesSecurity__ZoneInterfaces" "intf" {
    resource_name = "TEST"
    name = "TEST1"
    name__1 = "ge-0/0/0"
    name__6 = "all"
    #name__9 = "all"
}

resource "junos-qfx_commit" "commit" {
    resource_name = "commit"
    depends_on = [
        junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf
    ]
}
```
#### Terraform init
```
root@compute21:/home/aprabh/junos-terraform/terraform_providers# terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of 10.85.47.161/ns/junos-qfx from the dependency lock file
- Using previously-installed 10.85.47.161/ns/junos-qfx v1.0.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

#### Terraform plan and apply
```
root@compute21:/home/aprabh/junos-terraform/terraform_providers# terraform plan
junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf: Refreshing state... [id=10.102.144.2_TEST]
junos-qfx_commit.commit: Refreshing state... [id=10.102.144.2_commit]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  ~ update in-place

Terraform will perform the following actions:

  # junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf will be updated in-place
  ~ resource "junos-qfx_SecurityZonesSecurity__ZoneInterfaces" "intf" {
        id            = "10.102.144.2_TEST"
        name          = "TEST1"
      ~ name__1       = "ge-0/0/0.0" -> "ge-0/0/0"
      + name__6       = "all"
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run
"terraform apply" now.
``` 

```
root@compute21:/home/aprabh/junos-terraform/terraform_providers# terraform apply
junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf: Refreshing state... [id=10.102.144.2_TEST]
junos-qfx_commit.commit: Refreshing state... [id=10.102.144.2_commit]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  ~ update in-place

Terraform will perform the following actions:

  # junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf will be updated in-place
  ~ resource "junos-qfx_SecurityZonesSecurity__ZoneInterfaces" "intf" {
        id            = "10.102.144.2_TEST"
        name          = "TEST1"
      ~ name__1       = "ge-0/0/0.0" -> "ge-0/0/0"
      + name__6       = "all"
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf: Modifying... [id=10.102.144.2_TEST]
junos-qfx_SecurityZonesSecurity__ZoneInterfaces.intf: Modifications complete after 3s [id=10.102.144.2_TEST]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

### Validate on vSRX
Once configured, each of the TF configured ones would appear as a group and an apply group would be made. 

```
root> show configuration groups TEST
security {
    zones {
        security-zone TEST1 {
            interfaces {
                ge-0/0/0.0;
            }
        }
    }
}
```

#### check netconf logs 
```
root> show log netconf.log | last
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.0</capability></capabilities></hello>]]>]]>
<?xml version="1.0" encoding="UTF-8"?>
<rpc message-id="626179a0-5b50-4df5-91de-c5bd8095e044" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get-configuration>
  <configuration>
  <groups><name>TEST</name></groups>
  </configuration>
</get-configuration>
</rpc>]]>]]>

Jul 20 18:44:32 [NETCONF] - [66340] Outgoing: <rpc-reply xmlns:junos="http://xml.juniper.net/junos/20.4R0/junos" message-id="626179a0-5b50-4df5-91de-c5bd8095e044" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing: <configuration xmlns="http://xml.juniper.net/xnm/1.1/xnm" junos:changed-seconds="1626806672" junos:changed-localtime="2021-07-20 18:44:32 UTC">
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:     <groups>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:         <name>TEST</name>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:         <security>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:             <zones>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                 <security-zone>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                     <name>TEST1</name>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                     <interfaces>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                         <name>ge-0/0/0.0</name>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                     </interfaces>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:                 </security-zone>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:             </zones>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:         </security>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing:     </groups>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing: </configuration>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing: </rpc-reply>
Jul 20 18:44:32 [NETCONF] - [66340] Outgoing: ]]>]]>
```
