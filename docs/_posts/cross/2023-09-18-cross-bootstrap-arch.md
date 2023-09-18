---
layout: post
title:  "Cross-distro and cross-arch bootstrapping of ArchLinux (ARM)"
date:   2023-09-18 13:00:00 +0800
categories: cross
---

## Background
After several months of maintaining [orangepi5-archlinuxarm](https://github.com/7Ji/orangepi5-archlinuxarm) , I grew tired of the old method which bootstraps an ALARM rootfs inside an QEMU-based ALARM build environment on the Ubuntu 22.04 Github Actions runner, which was not only in-efficient but also a mess of mixture of 3 systems' permissions and filesystem structure.

Thus, the project switched to a nightly release model which utilizes [cross_nobuild.sh](https://github.com/7Ji/orangepi5-archlinuxarm/blob/232199e545143eee4efa57e7e9163d9d836a3f77/cross_nobuild.sh) to simply bootstrap on the x86-64 Ubuntu 22.04 host environment. The new builder took some effort as it is not only cross-distro but also cross-architecture, **and it did not use any pre-built rootfs archive at all, just like `pacstrap`** -- What was worked around as two seperate problems is now one single, big problem to solve. And I find that experience valuable to document.

_Although the main purpose behind this was to bootstrap ArchLinux ARM, it is also useful for bootstrapping any Arch-based distros from any architrecure, even Arch itself from ALARM._

## Host
The expected host environment is x86-64 Ubuntu 22.04, same as the main distro used on Github Actions. If you're using Arch itself then some of the following parts could be skipped. 

Although the distro where I figured out most of the process was also Arch itself, I don't think the steps on Arch itself would need to be documented, as you already have some very convenient scripts to use provided by `arch-install-scripts`.

Do note that the host **must be either a VM or an actual box**, I didn't test these inside a systemd-nspawn container, a Docker container, nor a LXC container. You will meet some challenges if you're running these containers due to the missing init.

## Dependency
### qemu-user-static
The only one hard dependency that's needed from the Ubuntu repo is `qemu-user-static`, which is needed to run necessary programs like those needed during Pacman's post-install scripts.
```
sudo apt update
sudo apt install qemu-user-static
```
### pacman-static
The other dependency we'll also need is `pacman`, or more precisely `pacman-static`, a build of `pacman` but with every library staticly linked, so it could run distro-independent. It's needed as we'll bootstrap the target from ground up with everything installed by `pacman`.

You can build it by yourself, but since there're already repos providing the pre-built binary we can save the effort by just downloading the binary.

An example of the repo is `archlinuxcn`, you can find the package on [opentuna's archlinuxcn mirror](https://opentuna.cn/archlinuxcn/x86_64), the latest one at the time of writing was [6.0.1-49](https://opentuna.cn/archlinuxcn/x86_64/pacman-static-6.0.1-49-x86_64.pkg.tar.xz), although the link would probably be dead when you read this blog in the future.

Download the package file and extract it, you would get a bunch of files but you only need `usr/bin/pacman`, move it to a place easier to call. You can also do all of these (downloading + extraction + mv) in one single step, e.g.
```
wget -O- https://opentuna.cn/archlinuxcn/x86_64/pacman-static-6.0.1-49-x86_64.pkg.tar.xz | tar -xOJ usr/bin/pacman-static | install -m 755 /dev/stdin pacman
```

Remember to test your pacman to confirm it works.
```
> ./pacman --version

 .--.                  Pacman v6.0.1 - libalpm v13.0.1
/ _.-' .-.  .-.  .-.   Copyright (C) 2006-2021 Pacman Development Team
\  '-. '-'  '-'  '-'   Copyright (C) 2002-2006 Judd Vinet
 '--'
                       This program may be freely redistributed under
                       the terms of the GNU General Public License.

```

## Target root
You'll need a dedicated mount point for the target root, it could be simply a folder bind-mounted to itself. The reason behind this is that a top-level mountpoint is always needed for many of the system utilities that'll run during the boostrapping.

You can also create a disk image, but we'll go the simple bind-mount route here just for demonstration:
```
sudo mkdir target
sudo mount -o bind target target
```
_Note the target root must be owned by root itself, so we used `sudo mkdir`_

And under the mount root, there're other subfolders needed:
```
root=target
sudo mkdir -p "${root}"/{dev/{pts,shm},etc/pacman.d,proc,run,sys,tmp,var/{cache/pacman/pkg,lib/pacman,log}}
sudo mount proc "${root}"/proc -t proc -o nosuid,noexec,nodev
sudo mount sys "${root}"/sys -t sysfs -o nosuid,noexec,nodev,ro
sudo mount udev "${root}"/dev -t devtmpfs -o mode=0755,nosuid
sudo mount devpts "${root}"/dev/pts -t devpts -o mode=0620,gid=5,nosuid,noexec
sudo mount shm "${root}"/dev/shm -t tmpfs -o mode=1777,nosuid,nodev
sudo mount run "${root}"/run -t tmpfs -o nosuid,nodev,mode=0755
sudo mount tmp "${root}"/tmp -t tmpfs -o mode=1777,strictatime,nodev,nosuid
```
Remember to also create and mount target/boot if you're bootstrapping into a disk image.

## Pacman config
Pacman won't run correctly if it misses the pacman config, found as `/etc/pacman.conf` under Arch.

We'll need two seperate configs, `pacman-loose.conf` which allows packages to be installed without signkey verification, and `pacman-strict.conf` which does not. The `loose` one is used to install the `base` package group and `archlinuxarm-keyring`, and the `strict` one is used to install everything else.

It is possible to omit the `strict` one, if you either install all pacakges in one step or chroot into the target root to install the remaining packages.
```
mkdir pkg
root=target
repo_url='http://mirror.archlinuxarm.org/aarch64/$repo'

pacman_config="
RootDir      = ${root}
DBPath       = ${root}/var/lib/pacman/
CacheDir     = pkg/
LogFile      = ${root}/var/log/pacman.log
GPGDir       = ${root}/etc/pacman.d/gnupg/
HookDir      = ${root}/etc/pacman.d/hooks/
Architecture = aarch64"

pacman_mirrors="
[core]
Server = ${repo_url}
[extra]
Server = ${repo_url}
[alarm]
Server = ${repo_url}
[aur]
Server = ${repo_url}"

echo "[options]${pacman_config}
SigLevel = Never${pacman_mirrors}" > pacman-loose.conf

echo "[options]${pacman_config}
SigLevel = Never${pacman_mirrors}" > pacman-strict.conf
```
You should notice that we explicitly set the `arch` as `aarch64`, this is due to the fact that `pacman` runs natively on `x86-64`. If not set then it would consider the `arch` to `x86_64`.

## Base packages
Install the `base` group and `archlinuxarm-keyring` into the target root

```
sudo ./pacman -Sy --config pacman-loose.conf --noconfirm base archlinuxarm-keyring
```

## Keyring
Before installing other packages, the keyring need to be initialized and populated, so the integrity could be checked. (It's also possible to go back and verify the packages in `base` group, but I'll omit that). 

First get into the target root:
```
sudo chroot target
```
Then init and populate the keyring:
```
pacman-key --init
pacman-key --populate archlinuxarm
```
Exit from the chroot to continue

## Other pacakges
Install other packages needed, but this time with the `strict` config, you'll need at least the kernel pacakge and the firmware package to ensure the target is bootable. And while we're at it, I recommend to install everything you need. E.g.
```
sudo ./pacman -Syu --config pacman-strict.conf --noconfirm vim nano sudo openssh linux-aarch64 linux-firmware
```
## Configuration
Follow [the Arch Wiki](https://wiki.archlinux.org/title/Installation_guide#Configure_the_system) to configure the system, except that you should use `chroot` itself to enter the target root.

## Finish
Due to `pacman-key` and `gpg-agent` expecting them running in a whole system, not in chroot, they would continue running, even after you existing from chroot. You'll need to kill them before umounting the target root:
```
sudo chroot target killall -s KILL gpg-agent dirmngr
```
Then you can umount the target root
```
umount -R
```
