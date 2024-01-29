---
layout: post
title:  "Non-nonflicting kernel and DTB layout on ARM for multi-boot"
date:   2024-01-29 12:00:00 +0800
categories: booting
---

Back in the early RPi days, there was not a single golden standard about how booting should be done on an ARM board and how kernels, initrds and DTBs shall be stored and loaded. Many chose to come up with their own booting script, and store them in a convenient way they see fit.

For many years, most distros would store their kernel, initrd and DTBs in a flat layout: kernel as `Image`, `Image.gz`, `zImage`, etc; initrd as `initrd`, `w`; dtb as `dtbs/vendor/board`. This is very simple and makes install/boot scripts easier to write. Later some would introduce "variable" supports in their scripts, from e.g. `uEnv.txt`, and make these changable, but most still install them to the path they were always using.

But a big problem is that every single distro could, even if they don't, use a totally different layout, and "common practice" followed by "most" distros never stops this from happening. After many years of the misary, u-boot finally decided enough is enough, and introduced "distro boot", which is, also many years before today, before even "distro boot" itself is deprecated.

The whole "distro boot" concept could be checked in older u-boot docs before it's deprecated, but the main idea is that, for each potentially bootable device, u-boot would try to `scan_dev_for_boot_part` to find a bootable partition, then `scan_dev_for_boot` to find either `extlinux/extlinux.conf`, or `boot.scr(.uimg)`, or `efi/boot/bootaa64.efi`, with an optional `/boot/` prefix (e.g., `extlinux.conf` would be looked up at `/extlinux/extlinux.conf` then `/boot/extlinux/extlinux.conf`). 

The first `extlinux` is my favourite one, as it uses an almost identical format as syslinux's extlinux, repurposing u-boot's pxe booting codes, it's therefore very easy to set up either by hand or by a script. A config for a single kernel could be written as the following:

```
LINUX   /vmlinuz-linux-aarch64-7ji
INITRD  /booster-linux-aarch64-7ji.img
APPEND  root=UUID=04af0995-78ca-4b66-8720-0459d81239e9 rootflags=compress=zstd:3,subvol=@ rw
FDT     /dtbs/linux-aarch64-7ji/amlogic/meson-g12a-s905l3a-cm311.dtb
FDTOVERLAYS /dtbs/overlays/meson-limit-emmc-frequency-100mhz.dtbo
```
Not hard to follow right? `LINUX` shall be the path to your kernel image, `INITRD` shall be the path to your initcpio image, `APPEND` shall be your kernel cmdline, `FDT` shall be the path to your DTB, and optionally `FDTOVERLAYS` shall be the list of paths to your DTB overlays.

This also could be written with different keys, like `FDT` could be swapped with `DEVICETREE`, etc. And all keys are case-insensitive, and order-insensitive, and here's a similar config with lower-case keys:

```
kernel /vmlinuz-linux-radxa-rkbsp5
initrd /initramfs-linux-radxa-rkbsp5.img
devicetreedir /dtbs/linux-radxa-rkbsp5
fdtoverlays  /dtbs/linux-radxa-rkbsp5/rockchip/overlays/rk3588-uart7-m2.dtbo
append   root=UUID=CHANGEME earlycon=uart8250,mmio32,0xfeb50000 console=ttyFIQ0 console=tty1 consoleblank=0 loglevel=0 panic=10 rootwait rw init=/sbin/init rootfstype=ext4 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1 irqchip.gicv3_pseudo_nmi=0 switolb=1 coherent_pool=2M
```

Another reason I prefer `extlinux` is that, as mentioned above, u-boot re-purposed PXE booting code for the job, which comes with menu support built-in, just like syslinux's own `extlinux`. So you could have a boot menu with multiple booting entries to choose from:

```
MENU TITLE Select the kernel to boot
TIMEOUT 30
DEFAULT linux-aarch64-7ji
LABEL   linux-aarch64-7ji
        LINUX   /vmlinuz-linux-aarch64-7ji
        INITRD  /booster-linux-aarch64-7ji.img
        FDT     /dtbs/linux-aarch64-7ji/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-rkbsp-joshua-git
        LINUX   /vmlinuz-linux-rkbsp-joshua-git
        INITRD  /booster-linux-rkbsp-joshua-git.img
        FDT     /dtbs/linux-rkbsp-joshua-git/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
```

This would print a menu on serial console and optionally video console if video driver built-in, like the following:

```
Select the kernel to boot
1. linux-aarch64-7ji
2. linux-rkbsp-joshua-git
```

Type a number and enter and the corresponding entry would be booted into, or you could wait for the 3-second timeout to pass (a unit in `TIMEOUT` ~= 0.1 second) and the default `linux-aarch64-7ji` entry would be chosen.

Here we already have multi-boot support! But let's take a step back to check how we store our kernel, initrd and dtbs:

```
> tree -L 3 /boot
/boot
├── booster-linux-aarch64-7ji.img
├── booster-linux-rkbsp-joshua-git.img
├── dtbs
│   ├── linux-aarch64-7ji
│   │   ├── amlogic
│   │   └── rockchip
│   └── linux-rkbsp-joshua-git
│       └── rockchip
├── extlinux
│   └── extlinux.conf
├── vmlinuz-linux-aarch64-7ji
└── vmlinuz-linux-rkbsp-joshua-git
```

If you use `Arch Linux` on x86_64 for daily driver you should be familiar with layout:

1. For each kernel, their kernel image shall be stored as `/boot/vmlinuz-[kernel name]`, so no kernel package would conflict with each other.
2. For each kernel, their initcpio shall be stored as `/boot/{initramfs|dracute|booster}-[kernel name]`, so no initcpio would conflict with each other.
3. **The important difference here**: For each kernel, their DTBs shall be stored under `/boot/dtbs/[kernel name]`, so no DTBs would confclit with each other.

This is very simple to be achieve, all you need to do is to install kernel and DTBs to a restricted path that's unique to each kernel package. But to my surprise not many distro would do this. A big reason I guess, is that most of them already have too many users, and changing those paths would be considered breaking change. So many just stick to the paths they've always been using since the early rpi days.

I introduced this to my [linux-aarch64-flippy-bin](https://github.com/7Ji-PKGBUILDs/linux-aarch64-flippy-bin/commit/3423660d6a8a64044aba562a24045747ef9e2f6a) on Nov 6, 2022 when creating my [amlogic-s9xxx-archlinuxarm](https://github.com/7Ji/amlogic-s9xxx-archlinuxarm) project, and thanks to the fact that the project was built from ground up, I didn't need to care about "existing" user base as I didn't have one. After that I've been always using that for all ALARM kernel packages I maintain. Maybe I'm the first kernel package maintainer to use this layout, but this is really just a small trick that anyone at any time could perform so there's nothing I want to argue here. A major point I want to make here is that both `amlogic-` and [orangepi5-archlinuxarm](https://github.com/7Ji/orangepi5-archlinuxarm) projects of mine have hundreds of users and such layout from day 1 never breaks.

At the end of the day, maybe this is not really needed for an end user who would only install and use one single kernel pacakges, but this would really be of good help for people like me who would constantly jump between different kernel packages they maintain. 

Let's end this with a config I'm using for one of my boards to test different kernels, I hope no one would need to jump among this many kernels every day : )

```
MENU TITLE Select the kernel to boot
TIMEOUT 30
DEFAULT linux-aarch64-7ji
LABEL   linux-aarch64-orangepi5
        LINUX   /vmlinuz-linux-aarch64-orangepi5
        INITRD  /booster-linux-aarch64-orangepi5.img
        FDT     /dtbs/linux-aarch64-orangepi5/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-aarch64-orangepi5-git
        LINUX   /vmlinuz-linux-aarch64-orangepi5-git
        INITRD  /booster-linux-aarch64-orangepi5-git.img
        FDT     /dtbs/linux-aarch64-orangepi5-git/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-aarch64-7ji
        LINUX   /vmlinuz-linux-aarch64-7ji
        INITRD  /booster-linux-aarch64-7ji.img
        FDT     /dtbs/linux-aarch64-7ji/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-joshua-git
        LINUX   /vmlinuz-linux-joshua-git
        INITRD  /booster-linux-joshua-git.img
        FDT     /dtbs/linux-joshua-git/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-rkbsp-joshua-git
        LINUX   /vmlinuz-linux-rkbsp-joshua-git
        INITRD  /booster-linux-rkbsp-joshua-git.img
        FDT     /dtbs/linux-rkbsp-joshua-git/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
LABEL   linux-rockchip-joshua-git
        LINUX   /vmlinuz-linux-rockchip-joshua-git
        INITRD  /booster-linux-rockchip-joshua-git.img
        FDT     /dtbs/linux-rockchip-joshua-git/rockchip/rk3588s-orangepi-5.dtb
        APPEND  root=UUID=0f113376-5f2c-4f96-ac28-99ba3b39673e rootflags=subvol=@,compress=zstd:3 rw
```