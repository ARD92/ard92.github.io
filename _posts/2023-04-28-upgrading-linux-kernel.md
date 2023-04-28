---
layout: post
title: Upgrading to linux mainline kernel
author: Aravind
tags: linux
---

Sometimes when we download cloud versions of image and run VMs, there might not be some kernel modules which we want. we would see errors like below 

```
root@jcnr2:~/platter# sudo modprobe mpls_iptunnel
modprobe: FATAL: Module mpls_iptunnel not found in directory /lib/modules/5.4.0-148-generic
```

To fix this, we need to upgrade kernel accordingly and below are the steps.

1. Download the kernel needed from the [link](https://kernel.ubuntu.com/~kernel-ppa/mainline/?C=N%3BO%3DD&ref=itsfoss.com)
2. Download stable versions instead of release candidates
3. install the .deb files 
    ```
    sudo dpkg -i *.deb
    ```
4. Reboot
    ```
    reboot
    ```
