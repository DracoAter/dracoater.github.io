---
layout: post
title: Windows Subsystem for Linux (WSL) and File Permissions using /etc/wsl.conf
author: Juri Timošin
tags:
- wsl
- windows
---

[1]: https://docs.microsoft.com/en-us/windows/wsl/release-notes#build-17063
[2]: https://docs.microsoft.com/en-us/windows/wsl/wsl-config#set-wsl-launch-settings

I encountered a problem with the Ubuntu inside Windows Subsystem for Linux. The problem is that
windows `C:` drive is mounted with root user.

<!--more-->

```bash
user@hostname:~$ ls -la /mnt/
total 0
drwxr-xr-x 1 root root 4096 Feb 11 09:36 .
drwxr-xr-x 1 root root 4096 Feb 11 09:36 ..
drwxrwxrwx 1 root root 4096 Feb 10 14:33 c
```

This causes later problems when you try to clone repository either with `hg` or `git`.

```bash
user@hostname:~$ hg clone <somerepo> --debug
[...]
applying clone bundle from <somerepo>.hg
abort: Operation not permitted: '<destination>/.hg/.dirstate-ICQalM~'
```

Also some other problems exist. If closing Ubuntu and killing the `init` process in Windows Task
Manager does not fix your problem, you can try using `/etc/wsl.conf` file. This is availabe since
[build 17063][1]. You need to create `/etc/wsl.conf` file inside your linux in WSL, and add the
default user and group ids for mount.

```
[automount]
options = "metadata,uid=1000,gid=1000" # in Ubuntu distro, the default user is created with uid=1000,gid=1000
```

There are also [other options available][2]. My `/etc/wsl.conf` now looks like that:

```
[automount]
enabled = true
options = metadata,uid=1000,gid=1000,umask=022
```
