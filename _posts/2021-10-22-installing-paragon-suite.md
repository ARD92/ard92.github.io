---
layout: post
title: Installing Paragon Automation Suite 
tags: Juniper
---
# Installing Paragon Automation Suite 
Paragon automation suite is a Juniper product which mainly comprises of Paragon planner (previous known as northstar planner ), Paragon Pathfinder (Previously northstar), Paragon Insights (Previously known as healthbot). All of them can be accessed in a single UI and be used for various use cases from Telemetry to WAN controller to provsion your RSVP-TE or SRTE LSPs.

## Prepare node to install

Usually you need to choose another VM to provision the containers on the primary and worker nodes. However in this case, the paragon package was loaded on primary VM to load the package. There was no separate VM used. 
Also, all the paragon VMs are in a private network hosted within the server. So in order to access the GUI, you will have to proxy or use SSH tunnelling. This is described in the later section. 

### Paragon Topology
```
                               192.168.122.0/24
    ------------+---------------+----------------+---------------+--------------
                |               |                |               |
                |               |                |               |
                |               |                |               |
           +----+----+        +-+-------+      +-+------+      +-+--------+
           | primary |        | worker1 |      |worker2 |      | worker3  |
    +------+         |    +---+         |   +--+        |  +---+          |
    |      |   .2    |    |   |    .3   |   |  |  .4    |  |   |    .5    |
    |      +----+----+    |   +----+----+   |  +----+---+  |   +------+---+
    |           |         |        |        |       |      |          |
    |           |         |        |        |       |      |          |
    |           |         |        |        |       |      |          |
    |           |         |        |        |       |      |          |   172.16.122.0/24
----+-----------+---------+--------+--------+-------+------+----------+--------
                |                  |                |                 |
                |                  |                |                 |
                |                  |                |                 |
                |                  |                |                 |   172.16.123.0/24
----------------+------------------+----------------+-----------------+---------
```



## Total setup including vMX
```
+-------+  +-------+     +--------+   +-------+  +--------+  +-------+  +--------+  +-------+
|       |  |       |     |        |   |       |  |        |  |       |  |        |  |       |
|  vmx1 |  | vmx2  |     |  vmx3  |   | vmx4  |  | vmx5   |  |  vmx6 |  | vmx7   |  | vmx10 |
|       |  |       |     |        |   |       |  |        |  |       |  |        |  |       |
+-----+-+  +---+---+     +----+---+   +----+--+  +----+---+  +----+--+  +----+---+  +--+----+
      |        |              |            |          |           |          |         |
      |        |              |            |          |           |          |         |
      |        |              |            |          |           |          |         |
      |        |              |            |          |           |          |         |
      +---+----+--------------+------------+----------+-----------+----------+---------+
          |                         10.102.144.0/24
          |
  iptables|
   host   |
   routing|
          |                           193.10.1.1/30
          +-----------------------------------------------------------------------------
                                  ^
                                  |
                                  |
                                  |   tunnel the 2 servers
                                  |
                                  |
                                  v    193.10.1.2/30
          -------------------------------------------------------------------------------
                                                                                         |
                                                                                         |  iptables
                                                                                         |   host
                                        192.168.122.0/24                                 |   routing
             ------------+---------------+----------------+---------------+--------------
                         |               |                |               |
                         |               |                |               |
                         |               |                |               |
                    +----+----+        +-+-------+      +-+------+      +-+--------+
                    | primary |        | worker1 |      |worker2 |      | worker3  |
             +------+         |    +---+         |   +--+        |  +---+          |
             |      |   .2    |    |   |    .3   |   |  |  .4    |  |   |    .5    |
             |      +----+----+    |   +----+----+   |  +----+---+  |   +------+---+
             |           |         |        |        |       |      |          |
             |           |         |        |        |       |      |          |
             |           |         |        |        |       |      |          |
             |           |         |        |        |       |      |          |
         ----+-----------+---------+--------+--------+-------+------+----------+--------
                         |                  |                |                 |
                         |                  |                |                 |
                         |                  |                |                 |
                         |                  |                |                 |
         ----------------+------------------+----------------+-----------------+---------

```
### Bring up the VMs on KVM using the below script 
```
root@ubuntu:/opt/aprabh# more ubuntu_vm_bringup_fix.sh
cd /root
sudo apt-get -y update; sudo apt-get -y install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils virt-manager
apt-get install -y net-tools tcpdump wget sshpass
https://linuxhint.com/libvirt_python/
apt install libguestfs-tools
mkdir -p /root/ubuntu-image
cd ubuntu-image
wget https://cloud-images.ubuntu.com/releases/focal/release/ubuntu-20.04-server-cloudimg-amd64.img
for s in s2 s3 s4 s5; do
cp /root/ubuntu-image/ubuntu-20.04-server-cloudimg-amd64.img /var/lib/libvirt/images/$s.qcow2
done
for s in s2 s3 s4 s5; do
qemu-img resize /var/lib/libvirt/images/$s.qcow2 +100G
done
mkdir -p /root/ubuntu-vm-bringup
for i in $(seq 2 5); do
cat > /root/ubuntu-vm-bringup/s$i-mgmt-interfaces.yaml <<EOF
network:
  ethernets:
    ens3:
      addresses: [192.168.122.$i/24]
      gateway4: 192.168.122.1
      dhcp4: no
      nameservers:
        addresses: [10.85.6.68]
      optional: true
    ens4:
      addresses:
        - 172.16.122.$i/24
        - 2001:db9:122::$i/64
      dhcp4: no
      nameservers:
        addresses: [10.85.6.68]
      optional: true
    ens5:
      addresses:
        - 172.16.123.$i/24
        - 2001:db9:123::$i/64
      dhcp4: no
      nameservers:
        addresses: [10.85.6.68]
      optional: true
  version: 2
EOF
done

#mv /root/ubuntu-vm-bringup/s2-mgmt-interfaces.yaml /root/ubuntu-vm-bringup/s1-mgmt-interfaces.yaml
#mv /root/ubuntu-vm-bringup/s3-mgmt-interfaces.yaml /root/ubuntu-vm-bringup/s2-mgmt-interfaces.yaml
#mv /root/ubuntu-vm-bringup/s4-mgmt-interfaces.yaml /root/ubuntu-vm-bringup/s3-mgmt-interfaces.yaml

#########
## For 4 servers
for s in s2 s3 s4 s5; do
virt-customize -a /var/lib/libvirt/images/$s.qcow2 \
--root-password password:Embe1mpls \
--hostname $s \
--run-command 'sed -i "s/.*PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config' \
--run-command 'sed -i "s/.*PermitRootLogin prohibit-password/PermitRootLogin yes/g" /etc/ssh/sshd_config' \
--upload /root/ubuntu-vm-bringup/$s-mgmt-interfaces.yaml:/etc/netplan/interfaces.yaml \
--run-command 'dpkg-reconfigure openssh-server' \
--run-command 'sed -i "s/GRUB_CMDLINE_LINUX=\"\(.*\)\"/GRUB_CMDLINE_LINUX=\"\1 net.ifnames=1 biosdevname=0\"/" /etc/default/grub' \
--run-command 'update-grub' \
--run-command 'apt-get purge -y cloud-init'
done


#Define bridge
##############

declare -A routerlink=( [s2]="k8s-data1 k8s-data2"
			[s3]="k8s-data1 k8s-data2"
			[s4]="k8s-data1 k8s-data2"
	       		[s5]="k8s-data1 k8s-data2")

for r in "${!routerlink[@]}"; do
    for b in ${routerlink[$r]}; do
        if [[ -z $(virsh net-list |grep "$b ") ]]; then
cat > /tmp/$b.xml <<EOF
<network><name>$b</name><bridge name='$b' stp='off' delay='0'/></network>
EOF
virsh net-define /tmp/$b.xml
virsh net-start $b
virsh net-autostart $b
rm -f /tmp/$b.xml
        fi
    done
done

#>> Define and start Ubuntu server
##################################
for s in s2 s3 s4 s5; do
virt-install --name $s \
--disk /var/lib/libvirt/images/$s.qcow2,size=100,format=qcow2 \
--vcpus 8 \
--cpu host-model \
--memory 20480 \
--network network=default,target=$s-ens3 \
--network network=k8s-data1,target=$s-ens4 \
--network network=k8s-data2,target=$s-ens5 \
--virt-type kvm \
--import \
--os-variant=${OS} \
--graphics vnc \
--serial pty \
--noautoconsole \
--console pty,target_type=virtio
done
ip addr add 192.168.122.1/24 dev virbr0
#sshpass -p Embe1mpls ssh -o StrictHostKeyChecking=no root@192.168.122.2
#sshpass -p Embe1mpls ssh -o StrictHostKeyChecking=no root@192.168.122.3
#sshpass -p Embe1mpls ssh -o StrictHostKeyChecking=no root@192.168.122.4
```

Copy the above script and save it as a ubuntu_vm_bringup.sh file 

```
chmod +x ubuntu_vm_bringup.sh
./ubuntu_vm_bringup.sh 
```
### Bring up vMXs 
This step can be performed later. Follow the link [vm topology builder](https://ard92.github.io/2021/06/18/juniper-vnf-topology-builder.html) to stand up vMXs accordingly. The topology file can be found in the [repo](https://github.com/ARD92/vm-topology-builder/tree/main/topologies/srv6-test)

### Install docker on all nodes 
Run the below commands on all primary and worker nodes  
```
 sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
 sudo apt-get update
 sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

### Create config
```
./run -c config-dir init
```

### Deploy the config 
This process takes about 30 mins 

```
./run -c config-dir conf
./run -c config-dir deploy
```

## Ceph and Rook challenges 
A separate partition is needed and it should be unformatted. 
There is a osd container which runs to completion which basically checks lsblk, hence having an unformatted partition is important.
```
root@s2:~# fdisk -l
Disk /dev/loop0: 61.85 MiB, 64835584 bytes, 126632 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop1: 67.26 MiB, 70516736 bytes, 137728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop2: 32.45 MiB, 34017280 bytes, 66440 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop3: 67.24 MiB, 70504448 bytes, 137704 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 102.2 GiB, 109735575552 bytes, 214327296 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 5BFA6485-7534-4F8B-9581-6EA45E5A559C

Device      Start       End   Sectors   Size Type
/dev/sda1  227328 214327262 214099935 102.1G Linux filesystem
/dev/sda14   2048     10239      8192     4M BIOS boot
/dev/sda15  10240    227327    217088   106M EFI System

Partition table entries are not in disk order.


Disk /dev/vda: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
notice /dev/vda. This should not be formatted into EXT4 or any file system. we just need to add and let it be so that ceph can utilize. please refer to [adding additional storage on KVM](https://ard92.github.io/2021/10/23/attaching-additional-storage-on-kvm-vm.html) to add. 

### To access Paragon

#### Check service IP allocation 
```
root@s2:~/Paragon21.2# kubectl get svc --all-namespaces | grep Load
ambassador             ambassador                            LoadBalancer   10.103.26.45     192.168.122.6   80:30436/TCP,443:31478/TCP,7804:30217/TCP,7000:31147/TCP   157m
ems                    ztpservicedhcp                        LoadBalancer   10.111.152.18    192.168.122.7   67:31687/UDP                                               145m
healthbot              hb-proxy-syslog-udp                   LoadBalancer   10.96.246.138    192.168.122.7   514:30464/UDP,11114:30971/UDP                              137m
healthbot              ingest-jti-native-proxy-udp           LoadBalancer   10.111.26.255    192.168.122.7   4000:32019/UDP,11111:32154/UDP                             129m
healthbot              ingest-snmp-proxy-preserve-sourceip   LoadBalancer   10.110.238.126   <pending>       162:30093/UDP                                              137m
healthbot              ingest-snmp-proxy-udp                 LoadBalancer   10.108.161.96    192.168.122.7   162:31260/UDP                                              137m
northstar              ns-pceserver                          LoadBalancer   10.111.95.24     192.168.122.8   4189:32730/TCP                                             132m
```

#### From laptop, create an SSH tunnel to the primary VM
192.168.122.6 is the IP used to expose the UI
10.85.47.167 is the server where all the paragon VMs were spun up
192.168.122.2 is the primary VM IP which is the k8s master 

```
ssh -L 8022:192.168.122.6:443 -J root@10.85.47.167 root@192.168.122.2
```

## Additional pointers
- make sure NTP is utilized. Chronyc is used by default and ntpd is not. Without this installation may fail 
- additional devices should be unformatted 
- at the least 20G ram, 100G HDD for root and 100G for ceph, 8 vCPU should be needed for VMs
