---
layout: post
title: Setting up HugePages in Ubuntu 
tags: linux
---

## Enable hugepages
```
sudo sysctl -w vm.nr_hugepages=102400

sudo reboot
```

### Meminfo
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

### Enable via Grub
A better way to allocate huge pages is to allocate via grub . Edit `/etc/default/grub`

```
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=10 iso
lcpu=2-23"
#GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1G hugepagesz=1G hugepages=5 transparent_hugepage=never
isolcpu=2-23"
#GRUB_CMDLINE_LINUX_DEFAULT="isolcpu=2-23"
GRUB_CMDLINE_LINUX=""
```

### Update grub
```
sudo update-grub

sudo reboot
```
Verify memory allocation using `/proc/meminfo`
