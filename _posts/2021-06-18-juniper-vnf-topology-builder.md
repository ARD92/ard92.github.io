---
layout: post
title: Topology builder for Juniper VNFs for fast functional testing  
author: Aravind 
tags: junos juniper kvm golang
---
Like many others, i too got bored of building topologies from scratch to do functional testing as and when any new feature releases on Junos. The process of editing multiple xml files to bring them on KVM and create bridges and use multiple VNFs was cumbersome. 
This topology builder is aimed at helping build random topologies for rapid bring up and help ease up the process of testing any network or even prepare for JNCIE :) . It does not support SRIOV and relies on virtio interfaces and linux bridges to make connections between the VNFs. 
Currently supported VNFs are vMX, vSRX and vQFX. 
This application works on top of KVM leveraging libvirt APIs to initate the topology

All one needs to do is define a yaml file in order to mention the connection between the nodes and the necessary system resources for the respective VNFs

## Sample topology yaml file
```
network_nodes:
  -
    name: "vmx1" # Name of the node 
    imagePath: "/home/aprabh/vmx_images/" # Path where the images would be present. The image would then be copied to build directory 
    reImage: "junos-vmx-x86-64-18.4R1-S7.1.qcow2" # RE image. For vSRX use only this since its a single image vNF 
    pfeImage: "vFPC-20200409.img" # PFE image . This would be application for vMX and vQFX only
    metadataImage: "metadata-usb-re.img" # Applicable only for vMX
    vhddImage: "vmxhdd.img" # Applicable only for vMX
    vnfType: "vmx" # Type can be either vmx, vqfx, vsrx
    re_memory: 1024 # Memory in Mbs
    pfe_memory: 4096 # Memory in Mbs
    re_cores: 1 # Number of cores for RE . The number CPU and cores should match
    pfe_cores: 4 # Number of cores for PFE. Not applicable for vSRX. only applicable for vMX and vQFX
    re_vcpu: 1 # Number of CPUs for RE
    pfe_vcpu: 4 # Number of CPUs for PFE
    re_port: 8610 # Console port for RE. you can telnet into this using telnet localhost <re_port>
    pfe_port: 8620 # Console port for PFE
    link: # these define the links that need to be created between this VNF and others. Make sure for the other direction,
             the same link name is used so that when bridges are created, the interfaces would be placed correctly 
      # name is user defined. This will be used to create the bridge name
      - name: "vmx1_vmx2"
        intf: "ge-0.0.0-vmx1"
        peerintf: "ge-0.0.0-vmx2"
  -
    name: "vmx2"
    imagePath: "/home/aprabh/vmx_images/"
    reImage: "junos-vmx-x86-64-18.4R1-S7.1.qcow2"
    pfeImage: "vFPC-20200409.img"
    metadataImage: "metadata-usb-re.img"
    vhddImage: "vmxhdd.img"
    re_memory: 1024
    pfe_memory: 4096
    vnfType: "vmx"
    re_cores: 1
    pfe_cores: 4
    re_vcpu: 1
    pfe_vcpu: 4
    re_port: 8611
    pfe_port: 8621
    link:
      - name: "vmx1_vmx2"
        intf: "ge-0.0.0-vmx2"
        peerintf: "ge-0.0.0-vmx1"
      - name: "vmx2_vsrx"
        intf: "ge-0.0.1-vmx2"
        peerintf: "ge-0.0.0-vsrx"
      - name: "vmx22_vsrx"
        intf: "ge-0.0.2-vmx2"
        peerintf: "ge-0.0.1-vsrx"
  -
    name: "vsrx"
    imagePath: "/home/aprabh/vsrx_images/"
    reImage: "junos-vsrx3-x86-64-20.4R1.12.qcow"
    pfeImage: ""
    metadataImage: ""
    vhddImage: ""
    re_memory: 4096
    pfe_memory:
    vnfType: "vsrx"
    re_cores: 1
    pfe_cores:
    re_vcpu: 1
    pfe_vcpu:
    re_port: 8612
    pfe_port:
    link:
      - name: "vmx2_vsrx"
        intf: "ge-0.0.0-vsrx"
        peerintf: "ge-0.0.1-vmx2"
      - name: "vmx22_vsrx"
        intf: "ge-0.0.1-vsrx"
        peerintf: "ge-0.0.2-vmx2"
```

## Download the application  
To access teh code, clone the directory from [here](https://github.com/ARD92/vm-topology-builder)

or 

```
git clone https://github.com/ARD92/vm-topology-builder
```
### build the application
```
docker run -itd golang:latest --name go -v ${PWD}:/usr/src/app bash 

go get <all packages> 

go build vm-topo.go
```

## Example to create/destroy or generate xml template for the  topology
```
./vm-topo create ipfabric-test.yaml

./vm-topo delete ipfabric-test.yaml

./vm-topo genxml ipfabric-test.yaml
```
## To generate topology diagram 
Instead of drawing out diagrams on power point, for the same topology build, you can generate the diagram as well using drawthenet.
[Drawthenet](https://github.com/cidrblock/drawthe.net)  is a very cool project to generate topology diagrams using yaml files. For the defined topology file you can generate anothe yaml file which is specific to drawthenet and render the image [here](go.drawthe.net). This makes it easier from drawing out images on power points. Once the yaml file is generated, simply copy and paste the content on the mentioned link. 
You can change the x and y coordinates accordingly to place your nodes. 

```
python generate_topology_diagram.py -t ipfabric-test.yaml
```
The saved yaml file would be of name drawthe_<topology file name>.yaml 

Hope this helps! 

