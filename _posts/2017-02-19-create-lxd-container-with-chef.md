---
layout: post
title: Create an Lxd Container With Chef
author: Juri Timošin
tags:
  - Lxd
  - Chef
  - Test Kitchen
---

Before going further to running integration tests on your cookbooks and roles we need to prepare a
bit by creating an Lxd container, where we would run our tests. This is a script that creats an Lxd
container with Chef installed and a running ssh server (required for Test Kitchen transport).

<!--more-->

{% highlight bash lineno %}
#!/bin/bash

codename=xenial

# download container
lxc init images:ubuntu/${codename}/amd64 u

# add network to the container
lxc network attach lxdbr0 u

lxc start u

# wait for network
sleep 5

lxc exec u apt-get -- update

# install ssh server
lxc exec u apt-get -- install openssh-server -y

# install Chef
lxc exec u apt-get -- install ruby ruby-dev make gcc -y
lxc exec u gem -- install chef -N
chef_version=`lxc exec u gem -- list ^chef$ -q`

# clean future image
lxc exec u apt-get -- clean
lxc exec u apt-get -- autoclean
lxc exec u find -- /var/lib/apt/ -type f -delete
lxc exec u rm -- -f ~/.bash_history

# create image
lxc stop u
lxc publish u --alias=kitchen-${codename}64 description="Ubuntu $codename (x64) with $chef_version"
lxc delete u
{% endhighlight bash %}
