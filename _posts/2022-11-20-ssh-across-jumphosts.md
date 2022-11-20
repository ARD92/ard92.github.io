---
layout: post
title: Setup ssh across multiple jumphosts
tags: linux
---

Sometimes when you need to ssh across multiple jump hosts openssh allows to use the "-J" flag. However this isn't that flexible when you have multiple jumphosts and you need to copy files or login to the machine. One of the easier way to do it is to use ssh proxy.

## Using SSH proxy and ssh config file 
Create a file as `~/.ssh/config`

```
vim ~/.ssh/config 
```

Copy the contents inside the file . Here `Host *` here for all hosts underneath, will use jumpost `jump1`. To login to `jump1` it uses `user1` and port `44028`. Further more hosts `host1`, `host2` and `host3` will use jumphost `jump1` and hence the proxy command `ProxyCommand` 

```
Host *
    ForwardAgent yes
    ServerAliveInterval 300
    ServerAliveCountMax 2

Host jump1
    Hostname 192.168.1.1
    User user1
    Port 44028

Host host1
    Hostname 192.167.1.1
    User user1
    ProxyCommand ssh jump1 -W %h:%p

Host host2
    Hostname 192.167.1.2
    User user1
    ProxyCommand ssh jump1 -W %h:%p

Host host3
    Hostname 192.167.1.3
    User user1
    ProxyCommand ssh jump1 -W %h:%p
```

Once saving the above file contents, you can ssh directly to the hosts

```
ssh host1 

ssh host2

ssh host3
```

In all the above 3 commands, it would ask for 2 passwords. 1st password would refer to jump host `jump1`. The second password refers to `host1/2/3` password  which would use username `user1`.

One can always setup SSH keys as well to avoid typing passwords. 

In order to SCP files, makes it much easier by just copying files to the host. Due to the above config, it takes care of proxying across the jump hosts. In the below example, localfile is copied to the destination host to the specified path.

```
scp <local file name> host1:<destination path>
```
