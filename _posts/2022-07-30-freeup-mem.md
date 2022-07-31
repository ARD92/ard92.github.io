---
layout: post
title: Free up memory in Linux 
tags: linux
---

1. Create a file freemem.sh 
2. Add permissions. `chmod +x freemem.sh`
3. execute as ./freemem.sh

```
root@ubuntu:~# more freemem.sh
sync; echo 1 > /proc/sys/vm/drop_caches
sync; echo 2 > /proc/sys/vm/drop_caches
sync; echo 3 > /proc/sys/vm/drop_caches
```

## Display memory 
```
root@compute21:~# free -h
              total        used        free      shared  buff/cache   available
Mem:           251G         80G         30G        8.1M        139G        168G
Swap:          255G         67M        255G
```
