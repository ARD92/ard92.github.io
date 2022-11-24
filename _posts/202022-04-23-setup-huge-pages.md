---
layout: post
title: Setting up Hugepages on linux env
tags: linux
---

When we get into performance testing of networks using dpdk, one of the things we run into is hugepages. By allocating higher huge page memory, we put less burden on the system to access page table entries. This offers sizes greater than 4KB which is the default page size. Finding an address is faster as fewer entries are needed in the TLB to provide memory coverage.

Read [here](https://www.ibm.com/support/pages/benefits-huge-pages#:~:text=With%20hugepages%2C%20finding%20an%20address,TLB%20to%20provide%20memory%20coverage.&text=Property%3A%20Hugepages%20are%20pinned%20to,in%20and%20out%20from%20RAM.) for more information 

## Enabling huge pages

To enable huge pages on linux. Modify the grub file under `/etc/default/grub` with below contents. 

### Edit grub file
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=10 isolcpu=2-23"
```

### update grub
```
sudo update-grub
```

### reboot the node
```
sudo reboot
```

## Alternate way to enable hugepages
 An alternative way to enable huge pages is to modify sysctl.conf file

### edit file
```
sudo vim /etc/sysctl.conf

vm.nr_hugepages = 10
```
This will add 10 huge pages each with 1G.

### update file
```
sysctl -p
```

## If you want to enable hugepages on the fly 

### Modify the below file for 2Mi 
```
vim /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

### Modify the below file for 1Gi 
```
vim /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
```

