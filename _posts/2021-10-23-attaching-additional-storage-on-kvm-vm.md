---
layout: post
title: Adding additional storage to KVM VMs 
tags: linux
---

# Adding additional storage on KVM

## Create the image 
```
sudo qemu-img create -f raw s2-attach.img 100G
```

## Copy and allocate blocks
```
sudo dd if=/dev/zero of=s2-attach.img bs=1M count=102400 status=progress
```
## Attach disk
Note that the target should be either vda, vdb ... sda or sdb may not get recognized.
```
virsh attach-disk {vm-name} --source <path to img> --target vda --persistent

Example
virsh attach-disk s2 --source /opt/aprabh/s2-attach.img --target vda --persistent
```

## Login to VM
once attached login to VM and perform the below steps

```
fdisk /dev/vda
```
type p for new partition . Default would 1 and hit enter followed by selection of blocks. hit enter. once complete press "w" to write and sync changes

## Format new partition
```
sudo mkfs.ext4 /dev/vda1

mkdir /vda1
mount /dev/vda1 /vda1
```

## add to fstab
```
vim /etc/fstab

/dev/vdb1    /disk2    ext4     defaults    0 0
```

