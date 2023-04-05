---
layout: post
title: Using dpdk pktgen as a software traffic generator 
tags: linux
---

## Setup pktgen 
I was always in need for traffic generator and i ended up writing my own packet crafter. However, was becoming too tedius to add new packet formats! 
In order for pktgen to be setup, either we can use it on baremetal/vm with sriov vfs being passed. However not all the times, we would have the flexibility of the NICs being connected. How about scenarios where you have multiple VMs and want to test something functionally and need a quick traffic generator ? In such cases, we can run packet gen on virtio interfaces 

### Bring up the dpdk packet gen VM 
use the script [here](https://github.com/ARD92/packet-gen-scripts/blob/main/setup_pktgen_vm.sh) to bring up the VM. The bash script just brings up 2 VMs with ubuntu

### Increase root partition size (optional)
sometimes the expanded disk doesnt show up on the vm when you run `dh -h`

you can do the below quick steps to increase the size
```
fdisk /dev/sda

press "p" for partition numbers
press "d" followed by "1" which is root partition
press "n" followed by hitting "return" so that default vals are accepted 
press "y" to accept change in signature
press "w" to sync the changes

resize2fs /dev/sda1
``` 

now validate the size and notice that it increased to 50G

```
root@pktgen2:~/dpdk_pktgen# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           394M  2.8M  391M   1% /run
/dev/sda1        51G  3.1G   48G   6% /
< ----- snipped ------- >
```

### Setup huge pages
Huge pages is important without which pktgen probably would not work. I have seen debugging references where we can pass a flag to skip but it is recommended to use huge pages 

Follow [here](https://ard92.github.io/2022/04/23/setup-huge-pages.html)  to update huge pages 

### Install dependencies 

#### If igb_uio module not found install the below 
```
sudo apt-get install -y dpdk-igb-uio-dkms
```

#### If uio_pci_generic module not found , install the below
```
sudo apt-get install -y linux-modules-extra-5.4.0-126-generic
```

#### Load the modules

```
modprobe modprobe igb_uio
modprobe uio_pci_generic
```

#### Verify
```
root@pktgen2:~/dpdk_pktgen# lsmod | grep uio
uio_pci_generic        16384  0
igb_uio                20480  0
uio                    20480  2 igb_uio,uio_pci_generic
```

At this stage all the dependencies are met and let us proceed with installation

### Clone the repo
```
git clone https://github.com/ARD92/dpdk_pktgen.git
```
Atoonk did a great job  of packaging it! it makes it seemless to install pktgen! Kudos to him! I was struggling to compile with all dependencies missing when I initially tried to setup pktgen. I forked the same repo.

### Modify the install-dpdk-pktgen.sh file 
Edit the file to add the pci address of the interfaces. In the above vm that you create, let us consider the interface `ens4` 

Gather the PCI bus info
```
root@pktgen2:~/dpdk_pktgen# lspci | grep Ether
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
00:04.0 Ethernet controller: Red Hat, Inc. Virtio network device
```

let us choose `00:04.0`. edit the file with this value for the export PCI_IF param.

```
export PCI_IF="0000:00:04.0"
```
save and quit the file.

### Install pktgen
```
chmod +x install-dpdk-pktgen.sh

./install-dpdk-pktgen.sh
```

This will take a bit of time to compile and install. At the end of the installation, just scroll up to ensure no error aborted the setup.

### verify if interface bound to dpdk 

Notice on the very first line `00:04:0` is bound to dpdk-compatible driver. we will use this to send packets out of. 
At this time since its bound, the interfaces would not show up on `ip link show`

```
root@pktgen2:~/dpdk_pktgen# /opt/dpdk-20.02/usertools/dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:00:04.0 'Virtio network device 1000' drv=igb_uio unused=vfio-pci,uio_pci_generic

Network devices using kernel driver
===================================
0000:00:03.0 'Virtio network device 1000' if=ens3 drv=virtio-pci unused=igb_uio,vfio-pci,uio_pci_generic *Active*

No 'Baseband' devices detected
==============================

No 'Crypto' devices detected
============================

No 'Eventdev' devices detected
==============================

No 'Mempool' devices detected
=============================

No 'Compress' devices detected
==============================

No 'Misc (rawdev)' devices detected
===================================
```

## Usage 
Now that pktgen is installed, let us look to see how to run the traffic.  You can use the example pkt file to start.

### Pkt file sample for single flow 
Save the below contents into a file pktgen.pkt 
```
stop 0
set 0 rate 0.1
set 0 ttl 10
set 0 proto udp
set 0 dport 81
set 0 dst mac 44:ec:ce:c1:a8:20
#set 0 src mac 00:52:44:11:22:33
#set 0 src mac 00:00:00:00:00:01
set 0 dst ip 10.99.204.8
set 0 src ip 10.99.204.3/30
set 0 size 64
```

### Run pktgen 
```
/opt/pktgen-20.02.0/app/x86_64-native-linuxapp-gcc/pktgen --file-prefix=pktgen -- -P -m 1.0 -f pktgen.pkt
```

Note that this guide talks about only virtio based interfaces and not for performance based. In case you need to generate more, provide enough resources to vm and pin the CPUs in the same numa for better performance. The .pkt files can be reused. 

once you run the above command you would see the below. use `start 0` to start traffic 

## Connect to a topology ?

Based on the above steps if you have pktgen running , you can pass this traffic to any VNF/CNF running on the server.
Just ensure the interface `ens4` far end on host is placed on same linux/ovs as the other VNF/CNF 

```
pktgen-data1		8000.52540008e7d7	no		pktgen-dta1-nic
							pktgen1-ens4
pktgen-data2		8000.525400d00b25	no		pktgen-dta2-nic
							pktgen2-ens4
```

## Additional references 

### Pkt file sample for range of flows
```
enable 0 range
range 0 dst mac 00:00:5e:00:01:00 00:00:5e:00:01:00 00:00:5e:00:01:00 00:00:00:00:00:00
range 0 src mac aa:bb:cc:dd:ee:50 aa:bb:cc:dd:ee:50 aa:bb:cc:dd:ee:50 00:00:00:00:00:00
range 0 dst ip 1.1.61.2 1.1.61.2 1.1.61.2 0.0.0.0
range 0 src ip 1.1.52.2 1.1.52.2 1.1.52.2 0.0.0.0
range 0 dst port 1024 1024 49151 1
range 0 src port 49152 49152 65535 1
range 0 size 64 64 64 0
set 0 size 64
set 0 count 0
set 0 rate 1
```

More samples can be found [here](https://github.com/ARD92/packet-gen-scripts)

### Running Lua scripts

Install below dependencies to run lua scripts. You can pass lua scripts just like how .pkt files are passed using the `-f` flag. 

```
sudo apt-get install liblua5.3-dev
sudo apt-get install liblua5.3-0
sudo apt-get install lua5.3
cd ~/Pktgen-DPDK
meson -Denable_lua=true build
ninja -C build
ldconfig
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH: /usr/lib/x86_64-linux-gnu/
```
### Using packetgen

- [https://pktgen-dpdk.readthedocs.io/en/latest/usage_eal.html](https://pktgen-dpdk.readthedocs.io/en/latest/usage_eal.html)
- [https://toonk.io/building-a-high-performance-linux-based-traffic-generator-with-dpdk/index.html](https://toonk.io/building-a-high-performance-linux-based-traffic-generator-with-dpdk/index.html)

### Finding sibling threadis
```
cat /sys/devices/system/cpu/cpu2/topology/thread_siblings_list
2,34

cat /sys/devices/system/cpu/cpu3/topology/thread_siblings_list
3,35
```
Here cpu 2 has sibling thread 34 and cpu3 has sibling thread 35. 

### STTY 
Once we run pktgen it might overwrite the screen. use `stty sane` to fix that

### Pktgen usage

### using it with Junos

When you want to pass traffic to vNFs such as vMX, vSRX. Make sure the mac address of the interfaces match with the destination mac on the .pkt file 

For example here the dmac address belongs to interface ge-0/0/0 of vMX 

#### Find mac address
```
show interfaces ge-0/0/0 extensive | match Curr
```

#### .pkt file
```
```

