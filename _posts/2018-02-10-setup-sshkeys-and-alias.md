---
layout: post
title: Setup ssh keys and bash profiles for easy access 
tags: linux
---

# Creating SSH keys 
Run the below commands and enter the required details. Once created it would be placed under path ~/.ssh/

```
ssh-keygen -t rsa
```

## Disable root login

Open the sshd_config file and update 

```
vim /etc/ssh/sshd_config

PermitRootLogin without-password
```
## copy the public key
```
ssh-copy-id -i ~/.ssh/id_rsa <user>@<ip>
```

## Restart service 

```
service sshd restart
```

Now you can start a session using ssh -i <file> <user>@<ip> to login to the node using the ssh keys instead of user/pass. 

# Creating bash alias 
create your aliases in a custom alias file under the path ~/.bash_aliases

```
more ~/.bash_aliases 
#My custom aliases
alias pi="ssh -i ~/.ssh/id_rsa <user>@<ip>"
```
Ensure the file bash_aliases is referenced in bashrc file. you would see something like the below

```
more ~/.bashrc
< ----- snipped ------ >
# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

```
once created, if you want to use the profile in the existing running terminal you would need to load the profile. Or simply exit the session and start a new one to load automatically 

```
. ~/.bash_profile
```

# References

* https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-2
