---
layout: post
title: Setup a pvt cloud for photos backup from android/ios
tags: linux
---

If you are looking for a google photos alternative on pvt cloud which can run on your raspberry pi or synology there are several alternatives. 

1. Next cloud
2. Lomorage
3. Photoprism 

Photoprism is really cool which allows for geo tagging, face recognition of people just like google photos. However it is intensive and can be skipped if you are only looking for photos auto backup and viewing. photo prism on pi may be intensive since indexing pictures itself will take a long time. Better to do it on a better performing platform.

In my use case, i just simply needed a photos backup without me wanting to copy files from phone or run a script from phone for automatic backups to my NAS running on raspberry pi. Lomorage is a very cool software which solves this problem.very light weight and runs on raspberry pi natively or as a docker container as well. I used the docker container model to keep things easier for myself as other services are running on the pi. 

## Installation
on your raspberry pi running ubuntu

### Download package
```
git clone https://github.com/lomorage/lomo-docker.git
```
### Build images
```
sudo docker pull lomorage/raspberrypi-lomorage:latest
```
### Edit env file
we need to add few IPs.
1. underlying interface which the pi talks to the outside world subnet is mentioned properly 
2. Gatway IP. This would be your router IP typically
3. container IP which lomorage would use. This would be an ipvlan type interface. This should be in the same subnet range
4. folder in which pictures would be stored

```
cd lomo-docker

## edit the env file
vim.tiny .env
```
### Start the service
```
docker-compose -f docker-compose.vlan.yml up
```
once service is up, use the phone app and navigate to `manage lomorage account`. The service would be discovered using mdns and you would be able to create a new userId. Until then you would see an error on docker compose as `msg="failed to register against cloud3.lomorage.com` or `no route to cloud3.lomorage.com`. 

### Verify
you can verify by going to `http://<lomorage-ip>:8000`

## References 
- ![lomorage-installation](https://docs.lomorage.com/docs/Installation/lomorage-service/)
- ![about-lomorage](https://lomorage.com/)
- ![photo-prism-lomorage](https://lomorage.com/blog/2022/02/11/photoprism/)

