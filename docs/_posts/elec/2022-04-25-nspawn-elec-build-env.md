---
layout: post
title:  "Set up a systemd-nspawn based LibreELEC/CoreELEC/EmuELEC build environment"
date:   2023-04-25 22:00:00 +0800
categories: elec
---

This blog documents a way to set up a systemd-nspawn based LibreELEC/CoreELEC/EmuELEC build environment

## What is systemd-nspawn
`systemd-nspawn` is a tool incorporated into systemd that can be used to spawn a new namespace in your currently running Linux host environment, which could be useful for installation/fixups (replacing `chroot`), or process isolation (replacing `docker`), or presumably-full-system environment isolation (replacing `lxc`)

## Why not just build in host environment?

The build environment, no matter what project it is for, is a complex thing to get right with. A lot of times you would want to use a specific version of compiler/library that's different from your host environment to build the project. 

This is true for all LibreELEC-derived projects: the developers at LibreELEC use Ubuntu20.04 as their development & build environment, and many things simply don't build for any other environment, not even Ubuntu22.04. CoreELEC and EmuELEC as derived projects down the line inherents most of the components and you should expect the same problems.

If you want to go with a dedicated Ubuntu20.04 installation, that is acceptable, but really brings more problems if you want the machine to do other things: For one thing Ubuntu20.04 is old, many popular server applications lags behind, from a security perspective you should not use it as your main OS due to outdated security components; for another thing deploying it on bare metal makes it very painful when you need to upgrade to a newer OS if the recommended build env is upgraded to something like Ubuntu22.04.

## Why containers?

A container to provide the build environment is easier to maintain compared to a dedicated system installation, more light-weight compared to a virtual machine, and more isolated and safer compared to a simple `chroot`ed or `PATH`-hacked rootfs folder from a Ubuntu20.04.

It both keeps your host environment clean and doesn't impact the performance if not running, and more importantly, doesn't expose more security concerns for your host environment.

## Why not Docker?

`Docker` is good for process/network isolation but really a bad idea for a build environment due to its nature for DevOps usage. I won't write much here to introduce `Docker` since you probably already know it, but the most important thing is that `Docker` uses a layered storage system that expects newer stuffs on the top of old stuffs. 

You would think it is not a big deal, yeah you could write a dockerfile to pack an image derived from the official Ubuntu20.04 image, with all of the dependency included. But what if the dependencies are changed due to newly introduced packages? After adapting your Dockerfile you would need then to either re-build the image from ground up again, wasting a lot of time and bandwidth, or add on another layer of the already in-efficient squashfs storage. 

You really wish to just get into the container and `apt upgrade/install` stuffs after doing that a lot of times. But since Docker is mostly aimed at process isolation, it's not straight-forward if you want to get into the container to modify things directly, especially when it is running. There is no better way than stopping the build and run a shell into it. 

Another reason against using Docker is that it is hardcoded to modify your firewall using iptables, this gets annoying fast if you're using nftables and you have a not-so-simple setup that involves a lot of traffics to moderate, which is not uncommon if you want to deploy the build env to a server on which you already run a lot of stuffs and possibly VMs.

## Benefit of using containers with init

Using LXC or systemd-nspawn, the containers you would run are almost full VMs in the sense they have their init process started, usually systemd, and therefore a lot of things can run side by side. You can expect the similar functionality of systemd in the container as in your host environment. This means you can do periodical builds using either `systemd.timer` units or `crontab` jobs, manage remotely with `ssh`, and easily update the env with `apt`. They all run in parallel. 

## Deployment

### 1. systemd-nspawn

First make sure `systemd-nspawn` is available in your host environment, if you're running a modern mainstream Linux distro, chances are your distro maintainers already packs the `systemd-nspawn` binary into your standard installation, alongside with the `systemd` package, type the following command to check some useful help messages:

```
systemd-nspawn --help
```

For Arch-derived distros you should get the help message directly since they pack `systemd-nspawn` as a part of the `systemd` pacakge which is then a member of the `base` package group, which you would definitely have installed.

For Debian-derived distros, you can install it from the `systemd-container` package:
```
apt update
apt install systemd-container
```

### 2. debootstrap

We will need to use `debootstrap`, Debian's official bootstrap tool, to bootstrap a rootfs of Ubuntu20.04.

For Arch:
```
pacman -Syu debootstrap
```

For Debian:
```
apt update
apt install debootstrap
```

Then we can simply `debootstrap` the a Ubuntu 20.04 rootfs into a folder `u20`, with `dbus` and `systemd-container` packages included so your host `systemd` can interact with the guest `systemd`
```
cd /var/lib/machines  # systemd's location to store nspawn containers
debootstrap --include=dbus,systemd-container --components=main,universe focal u20 http://archive.ubuntu.com/ubuntu
```
_You can replace `u20` with another name you like_

_You can replace `http://archive.ubuntu.com/ubuntu` with a url of a mirror site near you, so download could be faster_

### 3. setup

First we would need to get into the containers directly to do some basic setups

Use the following command to get a root shell inside the target rootfs, it's not a container yet. It's similar as a `chroot` with a few extra operations so you can also use `arch-chroot`
```
systemd-nspawn u20
```

Change the root password
```
passwd
```

Install packages
```
apt update
apt install \
    vim nano `# Text editting` \
    build-essential `# Basic build dependency` \
    git `# Version control system` \
    ssh `# Remote management` \
    screen tmux `# Multi session` \
    xfonts-utils `# Build system needs bdftopcf, mkfontdir and mkfontscale` \
    xsltproc `# Build system needs xsltproc` \
    default-jre `# Build system needs java` \
    libxml-parser-perl `# Build system needs perl::XML::Parser` \
    libjson-perl `# Build system needs perl::JSON` \
    libparse-yapp-perl `# Build system needs perl::Parse::Yapp::Driver` \
    libncurses5-dev `# Build system needs /usr/include/ncurses.h` \
    bc `# Build system needs basic calculator` \
    gawk `# Build system needs gawk` \
    zip unzip zstd lzop `# Build system needs them for compression` \
    patchutils `# Build system needs lsdiff for patching` \
    gperf `# Build system needs gperf for hasing` \
    golang `# Missing dep for at least one of the package` \
    zlib1g-dev `# Missing dep for at least one of the package`
```

Configure network, adapt and write the following content to `/etc/systemd/network/20-wired.network` in the container, or `/var/lib/machines/u20/etc/systemd/network/20-wired.network` if you want to operate in the host env. Replace the address according to what you want.

```
[Match]
Name=host0

[Network]
Address=192.168.17.20
Gateway=192.168.17.1
DNS=192.168.17.1
```

Enable `systemd-networkd`, `systemd-resolved` and `sshd`:
```
systemctl enable --now systemd-{network,resolve}d sshd
```

Create a non-root user:
```
useradd -g sudo builder -s /bin/bash -m
passwd builder
```

Now you can get out from the chroot and set the network for the container in your host environment, adapt and write the following content as `/etc/systemd/nspawn/u20.nspawn`:

```
[Network]
Bridge=bridge0
```

This will tell systemd-nspawn to init a virtual nic between the host and the container, with one end connected to the `bridge0` network bridge on host and another end available inside the container. This makes it easy to access the resource outside of the container. (If not set like this, the container's network will be host-only). 

For how to set up a bridge, you can check the folder `rz5` in my repo [systemd-networkd.conf.d](https://github.com/7Ji/systemd-networkd.conf.d). I've used vlans to seperate the traffic, and containers like this will be connected to a different vlan than my home traffic.

## Management

Use the following command to start the container:
```
machinectl start u20
```

If you want it to start on boot, you can enable it and the `machines.target` systemd unit:
```
machinectl enable u20
systemctl enable machines.target
```

Since we've enabled sshd in the container, you can just `ssh` into it to build or manage stuffs. You can also use one of the following built-in methods of `machinectl` to get into a running container:

To get a shell of a certain user directly:
```
machinectl shell [user@]u20 # write only u20 to get root shell
```

To get a login shell, so you can login as anybody:
```
machinectl login u20
```