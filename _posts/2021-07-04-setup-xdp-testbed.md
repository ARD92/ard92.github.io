---
layout: post
title: Setup testbed for testing XDP & ebpf 
tags: linux
---

## Use the below shell script to spin up a VM on KVM as the test bed
```
more xdp-test.sh

sudo apt-get -y update; sudo apt-get -y install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils virt-manager
apt-get install -y net-tools tcpdump wget sshpass
https://linuxhint.com/libvirt_python/
apt install libguestfs-tools
wget https://cloud-images.ubuntu.com/releases/focal/release/ubuntu-20.04-server-cloudimg-amd64.img
cp ubuntu-20.04-server-cloudimg-amd64.img /var/lib/libvirt/images/xdp.qcow2
qemu-img resize /var/lib/libvirt/images/xdp.qcow2 +100G
cat > /root/ubuntu-vm-bringup/mgmt-interfaces.yaml <<EOF
network:
  ethernets:
    ens3:
      addresses: [192.168.2.10/24]
      gateway4: 192.168.2.254
      dhcp4: no
      nameservers:
        addresses: [8.8.8.8]
      optional: true
  version: 2
EOF

virt-customize -a /var/lib/libvirt/images/xdp.qcow2 \
--root-password password:xdp123 \
--hostname xdp \
--run-command 'sed -i "s/.*PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config' \
--run-command 'sed -i "s/.*PermitRootLogin prohibit-password/PermitRootLogin yes/g" /etc/ssh/sshd_config' \
--upload /root/ubuntu-vm-bringup/mgmt-interfaces.yaml:/etc/netplan/interfaces.yaml \
--run-command 'dpkg-reconfigure openssh-server' \
--run-command 'sed -i "s/GRUB_CMDLINE_LINUX=\"\(.*\)\"/GRUB_CMDLINE_LINUX=\"\1 net.ifnames=1 biosdevname=0\"/" /etc/default/grub' \
--run-command 'update-grub' \
--run-command 'apt-get purge -y cloud-init' 

virt-install --name xdp \
--disk /var/lib/libvirt/images/xdp.qcow2,size=100,format=qcow2 \
--vcpus 3 \
--cpu host-model \
--memory 4096 \
--network=bridge=virbr0,target=ens3 \
--virt-type kvm \
--import \
--os-variant=${OS} \
--graphics vnc \
--serial pty \
--noautoconsole \
--console pty,target_type=virtio
ip addr add 192.168.2.254/24 dev virbr0
```

## start the VM and login
```
./xdp-test.sh 

ssh root@192.168.2.10 
password: xdp123
```
## Grow the partition size.
By default the cloud image is 2G only. Notice that 100G is the disk size mentioned. In order to grow follow the below

```
root@xdp:~# fdisk --list

root@xdp:~# fdisk /dev/vda

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

GPT PMBR size mismatch (4612095 != 209715199) will be corrected by write.
The backup GPT table is not on the end of the device. This problem will be corrected by write.

**At this point type "n" and leave to defaults hitting enter** 

Command (m for help): n
Partition number (2-13,16-128, default 2):
First sector (4612063-209715166, default 4612096):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4612096-209715166, default 209715166):

Created a new partition 2 of type 'Linux filesystem' and of size 97.8 GiB.

**at this point type "w" to save the partition**

Command (m for help): w
The partition table has been altered.
Syncing disks.

**Validate if the new partition has been created. Check /dev/vda2**

root@xdp:~# fdisk --list
Disk /dev/loop0: 67.58 MiB, 70848512 bytes, 138376 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop1: 55.43 MiB, 58114048 bytes, 113504 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop2: 32.28 MiB, 33845248 bytes, 66104 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: D2A0C4E2-A5C8-4762-9CC0-47B0A4F00CA4

Device       Start       End   Sectors  Size Type
/dev/vda1   227328   4612062   4384735  2.1G Linux filesystem
/dev/vda2  4612096 209715166 205103071 97.8G Linux filesystem
/dev/vda14    2048     10239      8192    4M BIOS boot
/dev/vda15   10240    227327    217088  106M EFI System

Partition table entries are not in disk order.
```

### Format the disk
```
root@xdp:~# mke2fs -j /dev/vda2
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 25637883 4k blocks and 6414336 inodes
Filesystem UUID: e11afc3d-3962-4268-ad99-030ab4ae730c
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done
Writing inode tables: done
Creating journal (131072 blocks): done
Writing superblocks and filesystem accounting information: done
```

### Mount the disk
```
mkdir -p vda2
mount /dev/vda2 vda2
```
### Add to fstab
```
root@xdp:~/vda2# more /etc/fstab
LABEL=cloudimg-rootfs	/	 ext4	defaults	0 1
LABEL=UEFI	/boot/efi	vfat	umask=0077	0 1
/dev/vda2 /vda2 ext3 defaults 1 2
```
### Verify the new partition
```
root@xdp:~/vda2# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           394M  972K  393M   1% /run
/dev/vda1       2.0G  1.3G  713M  65% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       68M   68M     0 100% /snap/lxd/20326
/dev/loop1       56M   56M     0 100% /snap/core18/2066
/dev/vda15      105M  7.9M   97M   8% /boot/efi
/dev/loop2       33M   33M     0 100% /snap/snapd/12159
tmpfs           394M     0  394M   0% /run/user/0
/dev/vda2        96G   61M   91G   1% /root/vda2
```
## resize root partition instead 
```
root@xdp:~# fdisk /dev/sda

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): d     <<<<<<<< delete partition 1 i.e. /dev/sda1
Partition number (1,14,15, default 15): 1

Partition 1 has been deleted.

Command (m for help): p
Disk /dev/sda: 102.2 GiB, 109735575552 bytes, 214327296 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: D2A0C4E2-A5C8-4762-9CC0-47B0A4F00CA4

Device     Start    End Sectors  Size Type
/dev/sda14  2048  10239    8192    4M BIOS boot
/dev/sda15 10240 227327  217088  106M EFI System

Command (m for help): n   <<<<<< create new partition with number 1. i.e./dev/sda1
Partition number (1-13,16-128, default 1): 1
First sector (34-214327262, default 227328):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (227328-214327262, default 214327262):

Created a new partition 1 of type 'Linux filesystem' and of size 102.1 GiB.
Partition #1 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: y

The signature will be removed by a write command.

Command (m for help): w <<<<<<< save
The partition table has been altered.
Syncing disks.
```

resize the partition

```
root@xdp:~# resize2fs /dev/sda1
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/sda1 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 13
The filesystem on /dev/sda1 is now 26762491 (4k) blocks long.
```

validate the size
```
root@xdp:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           394M  968K  393M   1% /run
/dev/sda1        99G  1.5G   98G   2% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop1       56M   56M     0 100% /snap/core18/2066
/dev/loop0       68M   68M     0 100% /snap/lxd/20326
/dev/loop2       33M   33M     0 100% /snap/snapd/12159
/dev/sda15      105M  7.9M   97M   8% /boot/efi
tmpfs           394M     0  394M   0% /run/user/0
```
## Install necessary packages
```
sudo apt-get update
sudo git clone https://github.com/xdp-project/xdp-tutorial.git 
sudo cd xdp-tutorial 
sudo git submodule update --init 
sudo git clone --recurse-submodules https://github.com/xdp-project/xdp-tutorial
sudo apt install clang llvm libelf-dev libpcap-dev gcc-multilib build-essential
sudo apt install linux-tools-$(uname -r) 
sudo apt install linux-headers-$(uname -r) 
sudo apt install linux-tools-common linux-tools-generic
```
