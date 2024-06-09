---
layout: post
title: Too many open files error
tags: linux
---

## Validate the limit
```
ulimit -n 
1024 
```

## Set new limit
```
ulimit -n 2000
```

## Validate
```
ulimit -a | grep open
```

## check inotify
```
sudo sysctl -a | grep inotify
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 1048576
user.max_inotify_instances = 128
user.max_inotify_watches = 1048576
```

## References
- https://www.tutorialspoint.com/fixing-the-too-many-open-files-error-in-linux#:~:text=The%20%22Too%20many%20open%20files%20error%22%20appears%20when%20the%20limit,Linux%20commands%20are%20case%2Dsensitive.
- https://linuxconfig.org/fixing-the-too-many-open-files-error-on-linux
