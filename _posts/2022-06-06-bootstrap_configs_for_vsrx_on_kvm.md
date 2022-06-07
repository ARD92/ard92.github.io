---
layout: post
title: Bootsrap configs to vSRX to start on kubevirt
tags: linux kubernetes junos
---

# Boot strap configs to vSRX spun up on kubevirt 

Currently the way to setup day 0 configs on cloud VMs is by using cloud init. Kubevirt provides ways to use start up script. However vSRX on KVM does not support cloud init with No cloud. In such scenarios, vSRX offers a way to bootstrap the config by attaching as a CDROM disk. In this post, you will see how to do that. 

## Create config file 

Create the day 0 config and store it as juniper.conf . Example as below.

```
system {
    host-name R0;
    root-authentication {
        encrypted-password "$6$rw0NmG2X$CiRhKBQuX7Hy8GJHRr5EDoGmnjov7D5PajpcR6wE.Vu51tGyZU07UOvkf4tgfe55uaH0PPGoB6l83RACOJvMu/"; ## SECRET-DATA
    }
    login {
        user test {
            full-name "Ansible Test";
            uid 9997;
            class super-user;
            authentication {
                encrypted-password "$6$EI42ANGw$sOWHHmTXUgLlrZWjdvKr4xfq2Ihz8g.h3/8pFYuo11SuOrN50XSqVOiNCMJEOS0wIGekmGFWoHo5cXktXnat11"; ## SECRET-DATA
                ssh-rsa "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDM5fpY4cqA4idcAHXYiZOO5YL4g/6"; ## SECRET-DATA
            }
        }
        user paragon {
            uid 2000;
            class super-user;
            authentication {
                encrypted-password "$6$woQBjSpJ$juVDqwHgXPWT3THQAPNa5fHmys36/U0ZDOjjtBe3P1hSbLJGPbYWk/1QYYrx84nHYDVdasEB.MRQ7BZz1fuNY0"; ## SECRET-DATA
            }
        }
    }
    services {
        ssh {
            root-login allow;
        }
        netconf {
            ssh;
        }
    }
    syslog {
        user * {
            any emergency;
        }
        file messages {
            any any;
            authorization info;
        }
        file interactive-commands {
            interactive-commands any;
        }
    }
}
interfaces {
    fxp0 {
        unit 0 {
            family inet {
                address 192.172.1.1/24;
            }
        }
    }
```

## Create an ISO image to bootstrap 

Now create an iso image with the config placed. 

### Create a directory
```
mkdir iso_dir
```

### Copy the created base config file
```
cp juniper.conf iso_dir
```

### Generate the ISO image
```
mkisofs -l -o config.iso iso_dir
```

### Create a Dockerfile to create container image

Save the below as as `Dockerfile`

```
FROM scratch
ADD config.iso /configpath/config.iso
```

#### Build the container image
```
docker build -t aprabh/vsrx:config .
```

#### Push to registry 

This is optional. If the worker nodes can reach the registry, this would be good since you do not need to manually copy the images to all the nodes.
```
docker tag <container name> <registry/image>
docker push <registry/image>
```

#### Copy the images to worker nodes 
If not pushing the images to registry, we need to manually copy the images to all nodes 

##### Save the image as tar ball
```
docker save --output vsrx_config.tar aprabh/vsrx:configpath
```

##### Copy the images 
```
scp vsrx_config.tar <user>@<workernode>:/var/tmp
```
Login to the worker node and load the image using
```
docker load -i vsrx_config.tar
```
Repeat the same on other nodes.

#### Validate the image 
```
root@k8s-master:~/kubevirt-bootstrap-cdrom# docker images | grep vsrx
aprabh/vsrx                                                                                     configpath           dd5ea2eee306   2 hours ago    358kB
```
### Modify the manifest file to use the config iso 

Notice the volume `config` attached as CDROM. This is important and if it is not as `CDROM`, the initialization of config will not happen and would not function as expected.
The disk itself is a containerDisk, so this need not be persistent. Once the configs are saved, it would save it on the boot disk against the PV. so unmounting the container disk post the first boot is perfectly fine. 
 
```
root@k8s-master:~/kubevirt-bootstrap-cdrom# more vsrx-kubevirt.yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vsrx-sriov
spec:
  running: True
  template:
    metadata:
      labels:
        kubevirt.io/vm: vsrx-sriov
    spec:
      domain:
        cpu:
          sockets: 1
          cores: 4
          threads: 1
        resources:
          requests:
            memory: 8Gi
            cpu: "4"
          limits:
            cpu: "4"
            memory: 8Gi
        devices:
          useVirtioTransitional: true
          disks:
            - name: boot
              disk:
                bus: virtio
            - name: config
              cdrom:
                readonly: true
          interfaces:
            - name: fxp
              sriov: {}
            - name: left
              sriov: {}
            - name: right
              sriov: {}
      volumes:
        - name: boot
          dataVolume:
            name: vsrx-sriov
        - name: config
          containerDisk:
            image: aprabh/vsrx:configpath
            path: /configpath/config.iso
      networks:
        - name: fxp
          multus:
            networkName: sriovnet-fxp
        - name: left
          multus:
            networkName: sriovnet-left
        - name: right
          multus:
            networkName: sriovnet-right
```

### Start the VM
```
kubectl apply -f vsrx_sriov.yml 
```

### Logs to validate during bootup

Notice that CDROM was attached during boot up and the configs will be loaded. 

```
Jun  6 19:54:59  R0 kernel: ISO is attached
Jun  6 19:54:59  R0 kernel: Found CDROM device.
Jun  6 19:54:59  R0 kernel: @ 1654545259 [2022-06-06 19:54:19 UTC] vmguest: setup ...
Jun  6 19:54:59  R0 kernel: @ 1654545259 [2022-06-06 19:54:19 UTC] vmguest: mounted /dev/cd0
Jun  6 19:54:59  R0 kernel: @ 1654545259 [2022-06-06 19:54:19 UTC] vmguest: no config found:
Jun  6 19:54:59  R0 kernel: total 3
Jun  6 19:54:59  R0 kernel: -r-xr-xr-x  1 0  wheel  1688 Jun  6 19:37 juniper.conf
Jun  6 19:54:59  R0 kernel: Checking platform support for: vsrx
Jun  6 19:54:59  R0 kernel: Attempting to add missing junos-appsecure-tvp
``` 

### Login to vSRX and check

Check the configuration post logging in. once booted up, it would ask for user/pass and would not be in amnesiac mode. This itself is an indicator that the initial configs were loaded. 

```
root@k8s-master:~/kubevirt-bootstrap-cdrom# virtctl console vsrx-sriov
Successfully connected to vsrx-sriov console. The escape sequence is ^]

root@R0> show configuration system | display set
set system host-name R0
set system root-authentication encrypted-password "$6$rw0NmG2X$CiRhKBQuX7Hy8GJHRr5EDoGmnjov7D5PajpcR6wE.Vu51tGyZU07UOvkf4tgfe55uaH0PPGoB6l83RACOJvMu/"
set system login user paragon uid 2000
set system login user paragon class super-user
set system login user paragon authentication encrypted-password "$6$woQBjSpJ$juVDqwHgXPWT3THQAPNa5fHmys36/U0ZDOjjtBe3P1hSbLJGPbYWk/1QYYrx84nHYDVdasEB.MRQ7BZz1fuNY0"
set system login user test full-name "Ansible Test"
set system login user test uid 9997
set system login user test class super-user
set system login user test authentication encrypted-password "$6$EI42ANGw$sOWHHmTXUgLlrZWjdvKr4xfq2Ihz8g.h3/8pFYuo11SuOrN50XSqVOiNCMJEOS0wIGekmGFWoHo5cXktXnat11"
set system login user test authentication ssh-rsa "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDM5fpY4cqA4idcAHXYiZOO5YL4g/6"
set system services ssh root-login allow
set system services netconf ssh
set system syslog user * any emergency
set system syslog file interactive-commands interactive-commands any
set system syslog file messages any any
set system syslog file messages authorization info
```

## Unmount the config ISO image 
Patch the manifest file to unmount the ISO image so next time it doesnt use it.

```
kubectl patch vm vsrx-sriov --type json -p='[{"op": "remove", "path":"/spec/template/spec/volumes/1"}, {"op": "remove", "path":"/spec/template/spec/domain/devices/disks/1"}]'

virtualmachine.kubevirt.io/vsrx-sriov patched
```

### Validate

Run the below to ensure the additional config disks are not present anymore. 

```
kubectl get vm vsrx-sriov -o=jsonpath="{.spec.template.spec.volumes}"
kubectl get vm vsrx-sriov -o=jsonpath="{.spec.template.spec.domain.devices.disks}"
```

Verify the xml file to ensure disc is unmounted 

#### Retrieve virt-launcher pod 

Find the pod and enter the pod 
```
root@k8s-master:~/# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vsrx-sriov-qrfxm   2/2     Running   0          22m

root@k8s-master:~/# kubectl exec -it virt-launcher-vsrx-sriov-qrfxm sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
Defaulted container "compute" out of: compute, volumeconfig, container-disk-binary (init), volumeconfig-init (init)
sh-4.4#
```

#### Check the XML created by kubevirt

Check if the config disc has been unmounted. 

```
sh-4.4# virsh list
 Id   Name                 State
------------------------------------
 1    default_vsrx-sriov   running

sh-4.4# virsh dumpxml default_vsrx-sriov

< ----------- Snipped ------------ >

<devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk' model='virtio-transitional'>
      <driver name='qemu' type='raw' cache='none' error_policy='stop' discard='unmap'/>
      <source file='/var/run/kubevirt-private/vmi-disks/boot/disk.img' index='3'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='ua-boot'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x03' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='qcow2' cache='none' error_policy='stop' discard='unmap'/>
      <source file='/var/run/kubevirt-ephemeral-disks/disk-data/config/disk.qcow2' index='1'/>
      <backingStore type='file' index='2'>
        <format type='raw'/>
        <source file='/var/run/kubevirt/container-disks/disk_1.img'/>
      </backingStore>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <alias name='ua-config'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

