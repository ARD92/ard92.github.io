---
layout: post
title: Setting up HugePages in Ubuntu 
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

### Enable unsafe IOMMU
```
sudo vim /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
sudo vim /sys/module/vfio_iommu_type1/parameters/allow_unsafe_interrupts
```

### validate Meminfo
```
root@k8s-master:~# more /proc/meminfo
MemTotal:       264069988 kB
MemFree:        236518876 kB
MemAvailable:   245510092 kB
Buffers:          461044 kB
Cached:          9531108 kB
SwapCached:            0 kB
Active:          9956476 kB
Inactive:        4624980 kB
Active(anon):    4592252 kB
Inactive(anon):     1444 kB
Active(file):    5364224 kB
Inactive(file):  4623536 kB
Unevictable:       18492 kB
Mlocked:           18492 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:               848 kB
Writeback:             4 kB
AnonPages:       4600248 kB
Mapped:          1132516 kB
Shmem:              6476 kB
KReclaimable:     853032 kB
Slab:            1748896 kB
SReclaimable:     853032 kB
SUnreclaim:       895864 kB
KernelStack:       57408 kB
PageTables:        83204 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    126792112 kB
Committed_AS:   239242576 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      434764 kB
VmallocChunk:          0 kB
Percpu:            92672 kB
HardwareCorrupted:     0 kB
AnonHugePages:     12288 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB
Hugetlb:        10485760 kB
DirectMap4k:     1085948 kB
DirectMap2M:    16607232 kB
DirectMap1G:    252706816 kB
```

## Verify if Intel VT-x is enabled ?

```
root@k8s-master:~# dmesg | grep -i DMAR-IR
dvuser57@ryp03:~/jcnr-R21.4/jcnr-vrouter$ sudo dmesg | grep -i DMAR-IR
[    0.599047] DMAR-IR: IOAPIC id 8 under DRHD base  0x957fc000 IOMMU 9
[    0.599050] DMAR-IR: HPET id 0 under DRHD base 0x957fc000
[    0.599052] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
[    0.601317] DMAR-IR: Enabled IRQ remapping in x2apic mode


root@k8s-master:~/jcnr-R21.4/jcnr-vrouter$ ls -l /sys/class/iommu/
total 0
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar0 -> ../../devices/virtual/iommu/dmar0
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar1 -> ../../devices/virtual/iommu/dmar1
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar5 -> ../../devices/virtual/iommu/dmar5
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar6 -> ../../devices/virtual/iommu/dmar6
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar7 -> ../../devices/virtual/iommu/dmar7
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar8 -> ../../devices/virtual/iommu/dmar8
lrwxrwxrwx 1 root root 0 Nov 23 22:15 dmar9 -> ../../devices/virtual/iommu/dmar9

root@k8s-master:~/jcnr-R21.4/jcnr-vrouter$ lscpu | grep VT-x
Virtualization:                  VT-x
```
