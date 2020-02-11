---
layout: post
title: Test-Kitchen with Vagrant and Virtual Box in Windows Subsystem for Linux (WSL)
author: Juri Timošin
tags:
- test-kitchen
- vagrant
- virtualbox
- wsl
- windows
---

[1]: https://www.vagrantup.com/docs/other/wsl.html
[2]: https://github.com/test-kitchen/kitchen-vagrant/blob/1c4cc1f4955003115e92fe3654a1e4590f6a9d18/lib/kitchen/driver/vagrant.rb#L249

I recently had to switch from Ubuntu to Windows on my workstation, but thankfully there is a
Windows Subsystem for Linux now there and I decided to setup my tools in Ubuntu inside Windows.
I will briefly describe the process and a solution for a small problem I encountered.

<!--more-->

## Install Required Software

To be able to start virtual machines with Test-Kitchen in WSL we need to install ChefDK and
Vagrant in WSL. After the installation create an environment variable:

```bash
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
```

You can add this code into `~/.bashrc`, it is set automatically on start. More on this in [Vagrant
Docs][1]. After that install VirtualBox in Windows.

## Test Current Configuration

I started testing if I could start the VM using Test-Kitchen by creating a simple `kitchen.yml` file.

```yaml
---
driver:
  name: vagrant

platforms:
  - name: ubuntu-18.04

suites:
  - name: default
```

This should download and start default ubuntu-18.04 image. After `kitchen create`, it downloaded
the image, but unfortunately, it couldn't start it and failed with the error:

```
vm:
* The host path of the shared folder is not supported from WSL. Host
path of the shared folder must be located on a file system with
DrvFs type. Host path: /home/${USER}/.kitchen/cache
```

It tries to create a shared folder although we did not ask for it. Lets look at the `Vagrantfile`
it created. The file is in `.kitchen/kitchen-vagrant/default-ubuntu-1804/Vagrantfile`.

```ruby
Vagrant.configure("2") do |c|
  c.berkshelf.enabled = false if Vagrant.has_plugin?("vagrant-berkshelf")
  c.vm.box = "bento/ubuntu-18.04"
  c.vm.hostname = "default-ubuntu-1804.vagrantup.com"
  c.vm.synced_folder ".", "/vagrant", disabled: true
  c.vm.synced_folder "/home/${USER}/.kitchen/cache", "/tmp/omnibus/cache", create: true
  c.vm.provider :virtualbox do |p|
    p.name = "kitchen-projects-default-ubuntu-1804-af21e153-1697-468a-adff-99007a6fd980"
    p.customize ["modifyvm", :id, "--audio", "none"]
  end
end
```

Indeed the main synced folder is disabled, but some other cache folder is there. The [source code
of kitchen-vagrant][2] has the answer:

```ruby
def safe_share?(box)
  return false if config[:provider] =~ /(hyperv|libvirt)/

  box =~ %r{^bento/(centos|debian|fedora|opensuse|ubuntu|oracle|amazonlinux)-}
end
```

They create a cache folder automatically, if your box is from *bento*. We need to either disable
this stuff or provide a better path. Adding this line to the `kitchen.yml` does the trick.

```yaml
driver:
  name: vagrant
  cache_directory: false # added line
```
