---
layout: post
title:  "Installation of ArchLinux ARM on an out-of-tree Amlogic device / 在不被官方支持的Amlogic设备上安装ArchLinux ARM"
date:   2022-11-08 21:10:07 +0800
categories: embedded
---

This article will guide you to install ArchLinux ARM on an out-of-tree Amlogic device in the ArchLinux way, which can grant you the maximum freedom as you can decide most of the details unlike guided installation where most of them are decided by the official maintainers like Manjaro, Ubuntu, etc.  
这篇文章会指导你如何在不被官方支持的Amlogic设备上以ArchLinux的方式安装ArchLinux ARM，这种安装方法能给你最大的自由，因为你可以决定大多数的细节，不像官方引导安装那样大多数细节被定死，比如Manjaro, Ubuntu那样

If you prefer a flash-and-boot experience, check my [amlogic-s9xxx-archlinuxarm][amlogic-s9xxx-archlinuxarm] project, where you could either build by yourself on ArchLinuxARM or download my pre-built images  
如果你更喜欢刷完就能启动，可以看我的[amlogic-s9xxx-archlinuxarm][amlogic-s9xxx-archlinuxarm]项目，你既可以在ArchLinuxARM上自己构建，也可以下载我构建好的镜像

## The Live environment / 安装环境

You'll need a live environment to do the installation, like how we need an ArchLinux installation ISO or other distros for the task for installation of ArchLinux on x86-64.   
你需要一个安装环境来进行安装，就像我们在x86-64的主机上安装ArchLinux时需要ArchLinux的安装镜像或其他发行版

The main purpose is to run several tools including Pacman (ArchLinux's package manager) **natively** on the machine itself, and potentially write stuffs to eMMC depending on the target drive which is almost impossible if you're not operating on the box (for several SBCs including Odroid N2, this should be possible as they can function as a card reader for eMMC; but not for most set-up boxes so we won't cover this).  
需要这个安装环境的主要目的是在目标机器本身上运行包括Pacman（ArchLinux的包管理器）在内的几个工具，并且也可能根据安装的目标驱动器需要向eMMC直写，而直写不直接在盒子本身上操作的话是几乎不可能的（对于一些包括Odroid N2在内的开发板来说，这是可能的，因为它们可以当成eMMC的读卡器使用；不过大多数的机顶盒都不可以这么做，所以我们不在此讨论此情况）

**You can certainlly use an x86-64 host as the live environment, if you're just installing on a SDcard; but you probably have to do all of these on the box itself if you want to install to eMMC    
如果只是要装到SD卡上的话，你确实可以用一个x86-84的宿主来充当这个安装环境；但只要你想安装到eMMC里，你基本就得在盒子本身上操作**

A pre-built image for Armbian (e.g. [ophub's Armbian][ophub's Armbian] or other well-known distros are good enough to give you this live environment, but even CoreELEC/EmuELEC/OpenWrt should do the work (yeah). If you have a bootable ArchLinux ARM this will be easier (where you probably want to migrate/duplicate instead of fresh installation) as some easy-to-use installation tools included in [arch-install-scripts][arch-install-scripts] package are right there (``pacstrap``, ``genfstab``, etc), otherwise you might need to steel these tools from ArchLinux Package mirror or an existing installation e.g. x86-64.  
预构建的Armbian（比如 [ophub的Armbian][ophub's Armbian]）或者其他知名发行版的镜像是充当这个安装环境的绝佳选择，不过即使你用CoreELEC/EmuELEC/OpenWrt也不会太妨碍整个安装的流程。如果你本身已经有了一个能启动的ArchLinux ARM那就更好了（不过这种情况下你可能想要迁移/克隆而不是全新安装），因为有的在[arch-install-scripts][arch-install-scripts]里的易于使用的安装工具唾手可得（``pacstrap``, ``genfstab``等），不然的话你可能需要从ArchLinux的镜像站或者是一个现存的比如x86-64平台上的安装里借用这些工具。

Refer to each project's documentation for their installation on SDcard/USBdrive seperatedly. I'll assume you can easily get this live environment up and not waste time wrting its steps here.  
参考各个项目的文档来分别了解怎么把它们安装到SD卡或者USB驱动器上。我就假设你能简简单单地搭建起这个安装环境，不会浪费时间写怎么做了。

## Partitioning / 分区

Like regular ArchLinux installation on x86-64, boot partition is optional depending on your booting environment setup, only the rootfs is always required (*its fs is not limited, though, but ext4 is recommended if you want to use a mainline u-boot as your primary bootloader (see [bootloader][bootloader] below) since then we can place everything including kernel, initramfs and fdt inside rootfs itself, which saves a lot of the precious space*). You can have seperate mount points for other folders under ``/`` but I don't want to waste much space on in-efficient partitioning so I'll stick with the single root idea.  
和x86-64平台上标准的ArchLinux安装一样，根据你的启动环境不同，boot分区是可选的，只有root分区是总是需要的（*不过，它的文件系统并不限定，但如果你想用主线u-boot作为你的主要引导程序的话，推荐ext4（参看下面的[引导程序][bootloader]章节），因为那样的话我们就能把包括内核，初始化内存盘，设备树在内的所有东西都放在root分区里面了*）。你可以让``/``下的其他文件夹都有各自的挂载点，不过我可不想在没效率的分区上浪费空间，所以本文贯彻一个root的思路

**If you're partitoning the eMMC, please check the [Partitioning of eMMC][part emmc] part below and make sure you know your on-board EPT layout if you're using the stock Amlogic bootloader, and make sure not to create MBR partitions that overlap with existing essential EPT partitions AND bootloader (usually 0~4M). And backup/restore the bootloader before and after partitioning. Partitioning the eMMC blindly may BRICK your device**  
**如果你要给eMMC分区的话，务必查看下面的[给eMMC分区][part emmc]章节，并且如果你使用Amlogic原厂的引导程序的话，请确定你清楚你的eMMC上的EPT分区表的布局，并且不要在已经存在EPT重要分区和引导程序（一般是0~4M）的位置创建MBR分区。并且在分区前后备份和恢复bootloader。盲目地给eMMC分区可能会导致你的设备变砖**

If you are using ``GNU/parted``, then the danger areas can be ignored easily like this, if e.g. the first 100MiB can't be touched:  
如果你用的是``GNU/parted``的话，那么危险的区域就可以像这样避开，比如说最前面的100MiB不能碰的时候：
```
parted -s /dev/emmc_device mkpart primary fat32 100MiB 164MiB
```
But if you are using ``fdisk``, then you need to calculate that partition's offset: ``100 MiB * 1024^2 Byte/MiB / 512 Byte/Sector = 204800 sectors``, so the partition needs to be created at 204800 sectors as offset, but the size could be a simple ``+64M``  
但如果你用的是``fdisk``，那么你就要计算分区的偏移：``100兆字节 * 1024^2字节/兆字节 / 512字节/扇区 = 204800 扇区``，那么分区就要创建在偏移204800扇区的位置，不过大小可以简单的``+64M``

If you'll need to use a mainline u-boot to boot, remember it'll scan for bootable partations, and will fallback to the first entry in the partition table if none of them is found. To mark a partition as bootable in a MBR partition table, either add a boot flag (with ``a`` in ``fdisk``, one per table), or set its type to ``EFI`` (with ``t`` then ``ef`` in ``fdisk``, every partition could be) or similar type.  
如果你要使用主线u-boot来启动的话，记住它会扫描可启动的分区，并且会在无结果的情况下回落到分区表里的第一个分区。要把一个MBR分区表里面的分区标记为可启动，要么添加一个启动标志（在``fdisk``里通过``a``命令，一个分区表里只能有一个），要么设置分区的类型为``EFI``（在``fdisk``里通过``t``命令，然后输入标识``ef``，所以分区都可以是）或者类似的类型

You have the following partition layouts to choose from, but they are really just recommendations (*Note none of them has a dedicated swap partition, which is usually not recommended for server use cases, but I like to do it in this way since I don't want to waste disk space on dedicated swap partition. You can create swapfile manually later. Also I'm strongly against using zram for swap, please just use swapfile*)  
你有以下可以选择的分区布局，不过它们都只是建议而已 （*注意，下面所有的布局里都没有单独的swap分区，对于当作服务器的使用情景来说这一般是不推荐的，不过我不喜欢在单独的swap分区上浪费空间，所以喜欢这么做。你可以之后手动创建swapfile。对了，我强烈不建议使用zram来作为swap，要用swap请直接用swapfile*）

### Seperated boot and root partition / 单独的boot和root分区
This can be used for almost all of the bootup environment mentioned below  
几乎可以适用于下文将谈及的任何引导环境

|part|fs|size|content|
|-|-|-|-|
|/boot|fat|>=64MiB|u-boot, kernel, initrd, fdt, syslinux config/uboot bootscript|
|/|any| >=2GiB|everything else

*``/`` can be any fs as long as it can be mounted by initrd*  
*``/`` 可以是任何文件系统，只要初始化内存盘能挂载它*

### Seperated u-boot and root partition / 单独的u-boot和root分区
This can only be used if you use a 2nd stage u-boot that supports loading files from an ext4 fs (mainline u-boot is often the case, see [Bootloader][bootloader] below)   
适用于使用可以从ext4文件系统加载文件的二阶段u-boot的情况（一般是主线u-boot，看下面的[引导程序][bootloader]章节）

|part|fs|size|content|
|-|-|-|-|
|u-boot|fat|>=1MiB|u-boot
|/|ext4| >=2GiB|everything else

*``/`` must be ext4, as it's easy to get a mainline u-boot that has a built-in ext4 driver (which is the defconfig for almost every device as of v2022.10), but not for btrfs, f2fs, etc.*   
*``/`` 必需是ext4，搞到内置ext4驱动的主线u-boot很简单（就v2022.10而言，几乎是所有设备的默认配置），btrfs, f2fs等可不然*

*``u-boot`` does not need to be mounted in userspace, its only purpose is that so the earlier stage u-boot can load this mainline u-boot from that vfat partition, which has nothing to do with the userspace. If you have to mount it, it's recommended to mount it at ``/boot/uboot``*  
*``u-boot`` 这个分区不需要在用户空间里挂载，它的唯一目的就是让更早一阶段的u-boot可以从这个vfat分区里面加载主线u-boot，和用户空间没有关系。如果一定要挂载的话，建议挂载到``/boot/uboot``*

### Single root partition / 单个root分区
This can only be used when a signed mainline u-boot is written to eMMC and replaced the on-board bootloader that came with Android (see the [Bootloader][bootloader] below)  
只有签名过的主线u-boot写到eMMC且覆盖安卓带的引导程序的情况下才可使用（看下面的[引导程序][bootloader]章节）

|part|fs|size|content|
|-|-|-|-|
|/|ext4|>=2GiB|everything

*``/`` must be ext4, as it's easy to get a mainline u-boot that has a built-in ext4 driver (which is the defconfig for almost every device as of v2022.10), but not for btrfs, f2fs, etc.*  
*``/`` 必需是ext4，搞到能内置ext4驱动的主线u-boot很简单（就v2022.10而言，几乎是所有设备的默认配置），btrfs, f2fs等可不然*


*Also note partitions in one table is not needed to be created on the same drive, they can be placed on different drives, as long as the fs containing kernels can be loaded by u-boot and the fs containing rootfs can be mounted by init*  
*也请注意同一张表里面的分区并不是一定要在同一个驱动器上创建，它们可以在不同的的驱动器上，只要装有内核的文件系统可以被u-boot加载，装有root的文件系统可以被init挂载*

## Formatting / 格式化
After you've partitioned the drive, the new partitions created should now be formatted with their corresponding fs. Refer to [File systems#Create a file system on Arch Wiki][Arch Wiki Doc Formatting] for how to create these fs. Examples:  
当你给驱动器分区以后，新创建的分区就需要以它们对应的文件系统来格式化。参考[Arch Wiki上的文件系统#创建文件系统][Arch Wiki Doc Formatting]来了解如何创建这些文件系统。例子：

To create an ext4 fs on the root partition  
在root分区上创建一个ext4文件系统
```
mkfs.ext4 -m 0 /dev/root_partition
```
To create a fat32 fs on the boot partition  
在boot分区上创建一个fat32文件系统
```
mkfs.vfat -F 32 /dev/boot_partition
```

## Mounting / 挂载
The target rootfs should be mounted so we could modify the content inside it, and the boot/uboot partition then optionally mounted if you've created it:  
目标根文件系统应该被挂载，这样我们能修改里面的内容，然后如果boot/uboot分区被创建了的话，你也应该挂载它们

To mount rootfs to ``/mnt``  
挂载根文件系统到``/mnt``
```
mount /dev/root_partition /mnt
```
To mount boot to ``/mnt/boot`` if you've created it  
如果你创建了boot分区，挂载它到``/mnt/boot``
```
mkdir -p /mnt/boot
mount /dev/boot_partition /mnt/boot
```

To mount uboot to ``/mnt/boot/uboot`` if you've created it to store a single mainline u-boot  
如果你创建了uboot分区来储存单独的主线u-boot，挂载它分区到``/mnt/boot/uboot``
```
mkdir -p /mnt/boot/uboot
mount /dev/uboot_partition /mnt/boot/uboot
```

## Installation / 安装
Make sure you've partitioned the target drive, this part assumes you've mounted the target rootfs to ``/mnt``, and optionally a potential boot partition to ``/mnt/boot`` and a potential uboot partition ``/mnt/boot/uboot``depending on your boot strategy  
请确保你已经给目标驱动器分区了，本部分假定你已经把目标根文件系统挂载到了``/mnt``，然后可选地把因启动策略需要而创建的boot分区挂载到了``/mnt/boot``，把uboot分区挂载到了``/mnt/boot/uboot``

### Bootstrap of the rootfs / root自举
To bootstrap the rootfs, we have two ways of doing this: either through ``pacstrap`` that comes in the [arch-install-scripts][arch-install-scripts] package on an existing ArchLinux ARM installation (or other distro with self-compiled [pacman][pacman] and stealling the ``pacstrap`` from another ArchLinux installation, e.g. your x86 main drive), or through extraction of a pre-populated rootfs archive  
要自举根文件系统，有两种方式：要么在现存的ArchLinux ARM安装上通过[arch-install-scripts][arch-install-scripts]包里带的``pacstrap``（或者通过在其他发行版上自己编译[pacman][pacman]然后再从另一个现存的ArchLinux安装里偷``pacstrap``，例如你的x86机子），要么通过从一个已经预先准备好的根文件系统归档里提取

#### Pacstrap
If both ``pacstrap`` and its dependencies are satisfied, a simple command like the following can give you a basic functionning rootfs:  
如果``pacstrap``和它的依赖都已经满足，那么像这样一条简单的命令就能给你一个基本的根文件系统
```
pacstrap /mnt base
```
You can add other packages to be bootstrapeed into the root, like text editors (e.g. vim, nano), remote management (e.g. ssh), permission management (e.g. sudo), etc. But even if you forget to install them here you can still install them later inside the root itself.  
你可以在这条命令里添加其他要被自举到这个根文件系统里的包，比如文本编辑器（例如vim, nano），远程管理（例如ssh），权限管理（例如sudo）等等。不过即使你在这里忘记安装它们了，等一下你在根文件系统里面也可以再安装

#### Rootfs archive / root档案包
ArchLinux ARM provides a periodically updated rootfs arhive for generic AArch64 targets that's available on many mirror sites, which can be either downloaded from [their main site][rootfs main], or from mirror sites such as [tuna][rootfs tuna] which is way more faster in China  
ArchLinux ARM为通用的AArch64目标提供定时升级的rootfs归档，可以通过许多镜像站下载，既可以从[它们的官网][rootfs main]，也可以从诸如[清华大学tuna][rootfs tuna]在内的镜像站下载，后者在中国明显更快

Assuming you have downloaded the archive as ``ArchLinuxARM-aarch64-latest.tar.gz``, it can then be extracted into the root with bsdtar and **root** permission (or with sudo):  
假定你已经把归档包下载为``ArchLinuxARM-aarch64-latest.tar.gz``，它可以被通过一条有**root**权限（自然也可以用sudo）的bsdtar命令来提取到根文件系统
```
bsdtar -C /mnt --acls --xattrs -xvpzf ArchLinuxARM-aarch64-latest.tar.gz 
```
### fstab / 文件系统表
You need to generate the fstab required by the target fs (``/etc/fstab``), which can either be written by yourself or generated with a script like ``genfstab`` from [arch-install-scripts][arch-install-scripts]. This script is architecture-independent so you can down load the package on your x86-64 Arch machine and steal it from ``/usr/bin/genfstab``, or just download and extract the package with ``tar`` and copy it.  
你需要生成目标系统需要的文件系统表（``/etc/fstab``），既可以自己手写也可以通过[arch-install-scripts][arch-install-scripts]里带的脚本``genfstab``来生成。这个脚本不依赖于特定架构，所以你可以在你的x86-64的Arch机器上安装这个包再从``/usr/bin/genfstab``来偷，或者干脆下载包以后用``tar``提取。

A command like this will generate the ``fstab`` with UUID as mount specifier, which will be much safer than fs label:  
像这样的一条命令就能用UUID作为挂载的标识符来生成``fstab``，比文件系统标签等方式简单地多
```
genfstab -U /mnt > /mnt/etc/fstab
```

If you write it manually, a line entry in ``fstab`` should look like this:  
如果手动填写的话，``fstab``里面的一行应该看起来像这样
```
UUID=5b6e6d63-686e-4ce0-a51d-22f8e68db73a / ext4 rw,noatime 0 1
```
Please refer to [fstab(5)][man fstab] for the exact format of ``fstab``, in most of the cases this should be in ``mount source``, ``mount point``, ``fs``, ``mount options``, ``dump``, ``passno``, seperated by space and/or tab  
请参考[fstab(5)][man fstab]来了解``fstab``的具体格式，多数情况下格式应该是``挂载源``，``挂载点``，``文件系统``，``挂载选项``，``转储``，``检查次数``这样子，以空格和/或换表符分割

### chroot / 变更根
From now on you'll need to enter the rootfs of the target installation to finish the installation. This relies on ``chroot``, and some extra operations are needed before and after the chroot. ``arch-chroot`` from the [arch-install-scripts][arch-install-scripts] can do these for you, and it's also architecture-independent, so you can just steal it from an existing x86-64 installation, or just extract it using ``tar`` from the raw package.  
从现在起你需要进入目标安装的根文件系统来完成安装。这依赖于``chroot``，以及前前后后的额外操作。[arch-install-scripts][arch-install-scripts]里的``arch-chroot``能帮你处理，并且也是无关于架构的，所以你可以从现存的x86-64安装里偷来用，或者干脆从包里用``tar``提取

#### arch-chroot / 半自动
If you have ``arch-chroot``, an easy command like this with root permission or sudo can put you into the target root, and clean everything up after you exit from it:  
如果你有``arch-chroot``，在有root权限或者是用sudo的情况下一条像这样的简单命令就能让你进入目标根，并且在你退出以后帮你清理
```
arch-chroot /mnt
```

#### manually / 手动
If ``arch-chroot`` is out of the window, you can still enter the chroot manually:  
如果``arch-chroot``不能用的话，你就得手动变更到这个根
```
mount --bind /mnt /mnt
cd /mnt
cp /etc/resolv.conf etc
mount -t proc /proc proc
mount --make-rslave --rbind /sys sys
mount --make-rslave --rbind /dev dev
mount --make-rslave --rbind /run run    # (assuming /run exists on the system)
chroot /mnt /bin/bash
```
And remember to ``umount`` these mount points under ``/mnt`` after you exited from the chroot (especially note ``/mnt`` itself is binded from ``/mnt``, this also needs umount)  
并且记得在你从这个变更过的根离开以后，解除``/mnt``下面这些挂载点的挂载（特别注意``/mnt``本身也是从``/mnt``挂载来的一个绑定挂载，也需要解除挂载）

### Initialization of Pacman keyring / 初始化Pacman密钥环
If you prepared the target root with the rootfs archive instead of ``pacstrap``, you'll need to initialize the Pacman, this is not needed for ``pacstrap`` since it'll do this for you:  
如果你是通过根文件系统归档而不是``pacstrap``来部署的目标根，你需要初始化Pacman，对于``pacstrap``部署的来说是不需要的，因为它会帮你初始化：
```
pacman-key --init
pacman-key --populate archlinuxarm
```
### Removal of the kernel and firmware / 移除内核和固件
If you prepared the target root with the rootfs archive instead of ``pacstrap``, you'll need to remove the kernel and the firmware as it is not usable for our device (We'll install our kernel and firmware from AUR later):  
如果你是通过根文件归档的形式安装的目标根，你需要移除内核和固件，因为它们对于我们的设备来说没有用（我们之后会通过AUR来安装我们的内核和固件）
```
pacman -Rcns linux-aarch64
```
This will remove ``binutils``, ``diffutils``, ``linux-firmware``, ``linux-firmware-whence``, ``mkinitcpio``, ``mkinitcpio-busybox``, ``linux-aarch64`` altogether, in which all except ``linux-aarch64``, ``linux-firmware`` and ``linux-firmware-whence`` will be reinstalled later as dependencies of our new kernel. If you want to be more specific, you can run the following command instead:  
这会把``binutils``, ``diffutils``, ``linux-firmware``, ``linux-firmware-whence``, ``mkinitcpio``, ``mkinitcpio-busybox``, ``linux-aarch64``一起移除，其中除了``linux-aarch64``, ``linux-firmware``和``linux-firmware-whence``外之后都会作为我们的新内核的依赖安装回来。如果你想要更精确一些的话，你可以运行下面这条命令:
```
pacman -R linux-aarch64 linux-firmware linux-firmware-whence
```
### Timezone / 时区
Setup the timezone via the following command (replace ``Asia/Shanghai`` with your area accordingly)  
通过下面的命令来设置时区（根据你的区域，替换``Asia/Shanghai``）
```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
Different from x86-64, usually you should not sync the time to RTC since most boxes won't come with RTC, the following command is only needed if your device has a RTC, which is barely the case and a premium that only some SBC has (e.g. Odroid N2+ comes with a RTC)  
和x86-64不同，一般你不需要把时间同步到RTC，因为大多数盒子都没有RTC，下面这条命令只有在你的设备有RTC的时候才需要，而这很罕见，只有少部分高贵的开发板比如Odroid N2+才有。
```
hwclock --systohc
```
### Localization / 本地化
Edit ``/etc/locale.gen`` and uncomment the locales you'll need to use, at least ``en_US.UTF-8 UTF-8`` is needed since it is a fallback locale before ``C``, and when things fall back to ``C`` many things will break  
编辑``/etc/locale.gen``并且取消你需要的语言的注释，至少``en_US.UTF-8 UTF-8``是需要的，因为它是``C``之前的回落语言，而当各种东西回落到``C``的时候很多东西都会坏掉

Run ``locale-gen`` to generate locales  
执行``locale-gen``来生成语言
```
locale-gen
```

Edit ``/etc/locale.conf`` and set the locale you'll need to use **before the space**, e.g. ``en_US.UTF-8 UTF-8`` was uncommented yet you should set ``LANG=en_US.UTF-8`` here if you want to use it as your locale  
编辑``/etc/locale.conf``并设置你需要使用的语言的**空格前面的部分**。比如，前面你取消了``en_US.UTF-8 UTF-8``的注释，这里需要设置的就是``LANG=en_US.UTF-8``
```
LANG=en_US.UTF-8
```

### Updating / 升级
*This step is put after localization because many packages will break when installing/upgrading if ``LANG`` is not set correctly (i.e. ``LANG=C``)  
这一步被放在本地化之后，因为如果``LANG``没有正确设置（也就是``LANG=C``），很多软件在安装/升级时会出错*
#### Pacman mirror / Pacman 镜像
ArchLinux ARM by default uses a GeoIP based mirror selecting "server" as the mirror for Pacman, the problem is that many mirror sites are not recorded by ArchLinux ARM maintainers so these good mirrors wouldn't be selected ever  
ArchLinux ARM默认情况是使用基于地理IP的镜像选择“服务器”作为Pacman的镜像，一个问题是很多镜像站没有被ArchLinux ARM的维护者收录，所以这些好镜像永远也不会被选中

If you are in China, then tuna is a good mirror site to use instead that is not recorded, edit ``/etc/pacman.d/mirrorlist``, comment the line  
如果你在中国，那么清华大学tuna就是一个不错的没有被收录的镜像站，编辑``/etc/pacman.d/mirrorlist``，注释掉这行
```
Server = http://mirror.archlinuxarm.org/$arch/$repo
```
And add a line  
并添加一行
```
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/$arch/$repo
```
#### Full upgrade / 完全升级
Since ArchLinux is a rolling release, never to a partial upgrade, run the following command to upgrade everything  
因为ArchLinux是一个滚动发行版，永远不要只局部升级，用下面这条命令来升级全部软件包
```
pacman -Syu
```
**You must do this now if you prepared the target rootfs by extraction of rootfs archive, since the archive is only updated per a few months  
如果你是通过提取根文件系统归档来部署这个根文件系统的，你现在必须升级，因为那个归档几个月才更新一次**

### Networking / 网络
#### hostname / 主机名
You need to set your hostname in ``/etc/hostname``, which should not be duplicated on your local network, note it can only has 15 characters and it's recommended to only use [0-9A-Z-_]. A hostname like ``hk17Ji`` will be seen as ``HK17JI`` by Windows hosts:  
你需要在``/etc/hostname``里设置主机名，这个主机名不应在你的本地网络中重复，注意主机名只能有15个字符，并且建议只使用数字、小写字母、横杠、下划线。像``hk17Ji``这样的主机名会被Windows主机识别成``HK17JI``
```
myhostname
```
#### hosts / 主机表
``/etc/hosts`` needs to be populated with well-known hosts, at least the following hosts are needed for some network programs to work:  
应该在``/etc/hosts``中填写众所周知的主机名，至少下面两条是必须的，没有它们一些网络程序无法正常工作

```
127.0.0.1       localhost
::1             localhost
```
#### network config / 网络配置
If you've prepared the root with ``pacstrap``, then no network manager is enabled; if you've prepared it by extracting the rootfs archive, then ``systemd-networkd`` and ``systemd-resolved`` are enabled by default, and configurations to start DHCP on interfaces ``eth0`` and ``en*`` are already there, you can skip this step  
如果你是通过``pacstrap``来部署这个根的，那么没有任何网络管理器被启用；如果你是通过提取跟文件系统归档来部署的，那么``systemd-networkd``和``systemd-resolved``默认就被启用，并且已经配置在网络接口``eth0``和``en*``上进行DHCP，你可以跳过这一步

A single configuration file (e.g. ``/etc/systemd/network/20-wired.network``) like the following is enough to set a static IP on ``eth0`` with network manager ``systemd-networkd`` that comes with ``systemd`` without any other software:  
像下面这样单单一个配置文件（比如``/etc/systemd/network/20-wired.network``）就足够通过``systemd``自带的网络管理器``systemd-networkd``配置在``eth0``上使用静态IP，不需要安装任何其他软件
```
[Match]
Name=eth0

[Network]
Address=192.168.7.13/24
Gateway=192.168.7.1
DNS=192.168.7.1
```
Or DHCP  
或者是动态IP
```
[Match]
Name=eth0

[Network]
DHCP=yes
DNSSEC=no
```
Remember to enable ``systemd-networkd`` and ``systemd-resolved``  
记得启用``systemd-networkd``和``systemd-resolved``
```
systemctl enable systemd-networkd.service systemd-resolved.service
```
#### User setup / 用户配置
For the user ``root``, you need to set up the password (If rootfs came from the archive, it'll have a default password ``root``, otherwise it's empty, but you should set the password nontheless)  
对于用户``root``，你需要设置密码（如果你是使用根文件系统归档部署的，那么``root``会有一个默认密码``root``，不然的话它是空的，但无论如何都应该设置密码）
```
passwd root
```
If you prepared the rootfs by extraction of the rootfs archive, it'll come with a user ``alarm`` with password ``alarm``, I don't like these kind of user accounts so I'll remove it:  
如果你的根文件系统是通过提取根文件系统归档来部署的，它会自带一个用户名``alarm``密码``alarm``的帐号，我不喜欢这种帐号，建议移除
```
userdel -r alarm
```
If you want to create a user that can use ``sudo``, remember to install the ``sudo`` pacakge if you haven't:  
如果你想创建一个能用``sudo``的帐号，记得安装``sudo``包，如果你早些时间没有的话
```
pacman -S sudo
```
And uncomment the line ``%wheel ALL=(ALL:ALL) ALL`` through ``visudo`` to allow the group ``wheel`` to ``sudo`` with their password:  
并通过``visudo``取消行``%wheel ALL=(ALL:ALL) ALL``的注释来允许组``wheel``的成员在输入密码以后运行``sudo``

*If you have installed VIM and don't like VI, you can link VI to VIM, by ``ln -sf vim /usr/bin/vi`` before ``visudo``  
如果你安装了VIM并且不喜欢VI，你可以把VI链接到VIM，在运行visudo前运行``ln -sf vim /usr/bin/vi``*
```
visudo
```
Create your own user, either with group ``wheel`` so it can use ``sudo``  
创建你自己的用户，要么把它添加到``wheel``组里，这样就能用``sudo``
```
useradd -g wheel -m nomad7ji
```
or not, if you prefer just switch to ``root``  
要么不添加，如果你喜欢直接切换到``root``的话
```
useradd -m nomad7ji
```
If you'll need to login as this user, set its password, otherwise you can only ``su`` to it from ``root``:  
如果你需要作为这个用户登录的话，设置它的密码，不然你只能从``root``用户通过``su``切换到这个用户：
```
passwd nomad7ji
```
#### SSH server / SSH服务器
*This step is put here after the creation of users because sshd by default only allows non-root users to login  
这一步被放在用户创建之后的原因是sshd默认只允许非root用户登录*

It's important to enable ssh server if you're planning to use the device as a server. If you prepared the target rootfs through extraction of the rootfs archive then this step can be skipped, as the ``ssh`` package is already installed and ``sshd.service`` is enabled by default  
如果你要把这个设备当成服务器使用的话，启用ssh服务器就很重要。如果你是通过提取根文件系统归档来准备的目标根文件系统，这一步可以跳过，因为其中``ssh``包已经预装且``sshd.service``默认已经被启用

Install the ``ssh`` package  
安装``ssh``软件包
```
pacman -S ssh
```
Enable the ``sshd.service`` systemd unit  
启用``sshd.service``单元
```
systemctl enable sshd.service
```
#### kernel / 内核
We'll need to deploy kernel and firmware through two AUR packages I've specifically made for Amlogic devices. A package for [kernel, header and dtb][AUR kernel] and a package for [firmware][AUR firmware] needs to be built. Both just fetches ophub's pre-built binaries and align them to the ArchLinux kernel style under the hood so it's not that slow. ``mkinitcpio`` can also play nicely with this kernel. Alternatively refer to [pre-built AUR packages][bin AUR] to just download them built by me without building  
我们需要通过两个我为Amlogic设备特别制作的AUR包来部署内核和固件。一个包是包含[内核，头文件和设备树][AUR kernel]的，一个包是包含[固件][AUR firmware]的。两个包都只是拉取ophub提前构建好的二进制并按照ArchLinux内核的风格来整理，所以不会太慢。``mkinitcpio``也可以和这个内核和平相处。也可以参见[提前打包好的AUR包][bin AUR]章节来直接下载打包好的包，而不需要构建

Install depedencies  
安装依赖
```
pacman -S base-devel git
```
Switch to the user you just created to build  
切换到你先前创建的用户来进行构建
```
su - nomad7ji
```
Build kernel package, which just fetches pre-compiled binaries from ophub's repo and align them to ArchLinux kernel's style, so it can co-exist as a unique kernel with ``linux``, ``linux-aarch64`` just like ``linux-lts``, ``linux-hardened``, and update its initramfs automatically with mkinitcpio hooks  
构建内核包，这个包只是从ophub的仓库拉取已经构建的代码，并且再修缮它，来符合ArchLinux内核的风格，这样它作为一个额外内核就像``linux-lts``, ``linux-hardended``那样可以和``linux``, ``linux-aarch64``共存，并通过mkinitcpio的钩子来自动更新初始化内存盘
```
git clone https://aur.archlinux.org/linux-aarch64-flippy-bin.git
cd linux-aarch64-flippy-bin
makepkg -s
cd ..
```
Build firmware pacakge, which just fetches pre-compiled binaries from ophub's repo and align them to ArchLinux firmware's style  
构建固件包，这个包只是从ophub的仓库拉取，再修改符合ArchLinux固件的风格
```
git clone https://aur.archlinux.org/linux-firmware-amlogic-ophub.git
cd linux-firmware-amlogic-ophub
makepkg -s
cd ..
```

Install these packages that has dependecies in-between in one Pacman command (``linux-aarch64-flippy-bin`` virtually provides ``linux-aarch64-flippy``, and depends on ``linux-aarch64-flippy-dtb``, which ``linux-aarch64-flippy-bin-dtb-amlogic`` virtually provides, and the kernel optionally depends on ``linux-firmware-amlogic-ophub``). The virtual kernel package is there so you can replace this package with a self-built ``linux-aarch64-flippy`` (not pushed to AUR) package instead:  
通过一条Pacman命令来同时安装包含有依赖关系的这些包（``linux-aarch64-flippy-bin``提供了``linux-aarch64-flippy``，并依赖于由``linux-aarch64-flippy-bin-dtb-amlogic``提供的``linux-aarch64-flippy-dtb``，以及可选依赖于``linux-firmware-amlogic-ophub``）。虚拟的内核包存在的目的是以后可以用自己构建的``linux-aarch64-flippy``包（没有发布到AUR）来平替它
```
pacman -U linux-aarch64-flippy-bin/linux-aarch64-flippy-bin-6.0.7-1-aarch64.pkg.tar.xz linux-aarch64-flippy-bin/linux-aarch64-flippy-bin-dtb-amlogic-6.0.7-1-aarch64.pkg.tar.xz linux-firmware-amlogic-ophub/linux-firmware-amlogic-ophub-20220916-1-aarch64.pkg.tar.xz
```

*``linux-aarch64-flippy-bin`` is a split-package, other unused packages ``linux-aarch64-flippy-bin-dtb-allwinner``, ``linux-aarch64-flippy-bin-dtb``， ``linux-aarch64-flippy-bin-headers`` are also built using the same ``PKGBUILD`` in the same ``makepkg`` session, you can install them if you need them  
``linux-aarch64-flippy-bin``是一个拆分包，其他没有用到的包``linux-aarch64-flippy-bin-dtb-allwinner``, ``linux-aarch64-flippy-bin-dtb``， ``linux-aarch64-flippy-bin-headers``也使用同一个``PKGBUILD``在同一次``makepkg``中创建，如果需要的话你可以安装它们*


## Bootloader / 引导程序
This part has been splitted to its own page, please refer to [Bootflow and configuration on Amlogic platforms][bootflow and config] instead  
这一部分已经被分割到单独的页面中，请参看[Amlogic设备上的启动流程和配置][bootflow and config]


## Initrd / 初始化内存盘
If your booting mechanism relies on ``booti`` instead of ``sysboot`` command to load the kernel, initrd and fdt under the hood, then the initrd must be packed as u-boot legacy ramdisk image, otherwise (mainline + syslinux config), you can skip this part  
如果你的启动方式依赖于``booti``而不是``sysboot``命令来加载内核，初始化内存盘和设备树，那么初始化内存盘必需打包为u-boot传统内存盘镜像；如果是主线+syslinux配置，你可以跳过这一部分

Use a simple command like this to convert a normal initramfs file to u-boot legacy ramdisk image:  
用一条像这样的简单命令来把普通的初始化内存盘文件转换成u-boot传统内存盘镜像
```
mkimage -A arm64 -O linux -T ramdisk -C gzip -n uInitrd -d /path/to/normal/initramfs /path/to/uboot/legacy/initrd
```

But since this is long and tedious, you can install my Pacman hook [uboot-legacy-initrd-hooks][AUR hook], which will generate the legacy initrd for you, alternatively refer to [pre-built AUR packages][bin AUR] if you don't want to build it:  
不过因为这条命令又臭又长，你可以安装我的Pacman钩子[uboot-legacy-initrd-hooks][AUR hook]，这个钩子会帮你生成传统内存盘镜像，也可以参见[预构建的AUR包][bin AUR]而不构建

*Build hook package  
构建钩子*

```
git clone https://aur.archlinux.org/uboot-legacy-initrd-hooks.git
cd uboot-legacy-initrd-hooks
makepkg -s
cd ..
```

Install the package with a simple Pacman command  
通过一条Pacman命令来安装这个包
```
pacman -U uboot-legacy-initrd-hooks/uboot-legacy-initrd-hooks-0.0.1-1-aarch64.pkg.tar.xz
```
This package is more than just two hooks, the script (``/usr/bin/img2uimg``) that's called by these hooks under the hood can be called by user directly. You can call it to conviniently convert a normal initramfs to a uboot legacy ramdisk using a command like this:  
这个包不只是两个钩子而已，这两个钩子里面调用的脚本（``/usr/bin/img2uimg``）可以直接被用户使用。你可以通过像这样的一条命令它来便捷地把一个普通的初始化内存盘转换为uboot传统内存盘：
```
img2uimg /folder1/folder2/initramfs.img
```
*the result uimg will be ``/folder1/folder2/initramfs.uimg``  
转换出来的uimg路径为``/folder1/folder2/initramfs.uimg``*

It can also convert multiple initramfs in one call:  
它也能在一次命令里转换多个初始化内存盘
```
img2uimg initramfs-linux-a.img initramfs-linux-b.img initramfs-linux-a-fallback.img initramfs-linux-b-fallback.img
```
If calling without argument, it will convert all linux kernels' initramfs that has a preset under /etc/mkinitcpio.d to uimg. It's recommended to run ``img2uimg`` once after you run ``mkinitcpio -P`` to update initramfs manually without invoking the hooks:  
如果没有没有任何参数，它会把所有在``/etc/mkinitcpio.d``下有预设的linux内核的初始化内存盘转化成uimg。建议在你不经过钩子，手动运行完``mkinitcpio -P``来更新初始化内存盘以后，运行一次``img2uimg``
```
mkinitcpio -P
img2uimg
```

## Extra: Partitioning of eMMC / 给eMMC分区

This part has been splitted to its own page, please refer to [Partitioning on Amlogic's proprietary eMMC partition table with ampart][ept with ampart]  
这一部分已经被分割到单独的页面中，请参看[使用ampart在Amlogic专有的eMMC分区表上分区][ept with ampart]

## Extra: Prebuilt AUR packages/提前打包好的AUR包

For all of the AUR packages mentioned in this article ([linux-aarch64-flippy-bin][AUR kernel], [linux-firmware-amlogic-ophub][AUR firmware], [uboot-legacy-initrd-hooks][AUR hook], [ampart][AUR ampart]), I've uploaded the pre-built packages to [my resource site][AUR bin mirror], you can use them directly if you trust me (but you shouldn't, better build them by yourself, you should only trust official repos on Linux):  
对于本文中提到的所有AUR包（[linux-aarch64-flippy-bin][AUR kernel], [linux-firmware-amlogic-ophub][AUR firmware], [uboot-legacy-initrd-hooks][AUR hook], [ampart][AUR ampart]），我都把构建好的包上传到了[我的资源站][AUR bin mirror]，如果你信任我的话（但你真的不应该，最好自己构建，在Linux上应该只信任官方的仓库）:

These are their sha256sums at the time this article is written:  
写本文时的这些包的sha256校验和
```
8bfc7590a9d7095f415c733a196cf1b3dc12099d59e8e70a7ecff236d557eb8c  ampart-git-0.1.1.r69.8b9fdb6-1-aarch64.pkg.tar.xz
f39ff6d34591ba09c53ae4f81d6556b56c921ee99306f113f87e0d25d242c495  linux-aarch64-flippy-bin-6.0.7-1-aarch64.pkg.tar.xz
39deed3115493575cc66b37da55bc531a0c6c11dd4316d7d1aff243bc420e997  linux-aarch64-flippy-bin-dtb-allwinner-6.0.7-1-aarch64.pkg.tar.xz
b83b7718e8d3c7a55660cdecbbb166591055f0523a805041b4b2c4e6a290c2fc  linux-aarch64-flippy-bin-dtb-amlogic-6.0.7-1-aarch64.pkg.tar.xz
d7cadd77505f5f5cee42848f7a3afd20e51d23ca0116fb16c6ddbf25d1d08bef  linux-aarch64-flippy-bin-dtb-rockchip-6.0.7-1-aarch64.pkg.tar.xz
49957e7d9b858b848991dd4d8f12c9943bb1940a071ce93dc17bd37cfb2732c0  linux-aarch64-flippy-bin-headers-6.0.7-1-aarch64.pkg.tar.xz
31543083749eaaed48fc6e5e61eeb79eb7798f3d6b7d4ef5122ff8237ae31cbc  linux-firmware-amlogic-ophub-20220916-1-aarch64.pkg.tar.xz
e535229316d23960e4e6588f6bf409410f9defccdc7367dca8c6a3cbaad78e30  uboot-legacy-initrd-hooks-0.0.1-1-aarch64.pkg.tar.xz
```

[bootloader]: #bootloader--引导程序
[bin AUR]: #extra-prebuilt-aur-packages提前打包好的aur包
[part emmc]: #extra-partitioning-of-emmc--给emmc分区

[bootflow and config]: ../11/amlogic-booting.html
[ept with ampart]: ../11/ept-with-ampart.html

[Arch Wiki Doc Formatting]: https://wiki.archlinux.org/title/File_systems#Create_a_file_system
[AUR ampart]: https://aur.archlinux.org/packages/ampart-git
[AUR bin mirror]: https://ee.fuckblizzard.com/AUR
[AUR firmware]: https://aur.archlinux.org/packages/linux-firmware-amlogic-ophub
[AUR hook]: https://aur.archlinux.org/packages/uboot-legacy-initrd-hooks
[AUR kernel]: https://aur.archlinux.org/packages/linux-aarch64-flippy-bin
[amlogic-s9xxx-archlinuxarm]: https://github.com/7Ji/amlogic-s9xxx-archlinuxarm
[amlogic-s9xxx-armbian]: https://github.com/ophub/amlogic-s9xxx-armbian
[ophub's Armbian]: https://github.com/ophub/amlogic-s9xxx-armbian/releases
[arch-install-scripts]: https://archlinuxarm.org/packages/any/arch-install-scripts
[man fstab]: https://man.archlinux.org/man/fstab.5
[pacman]: https://archlinuxarm.org/packages/aarch64/pacman
[rootfs main]: http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
[rootfs tuna]: https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/os/ArchLinuxARM-aarch64-latest.tar.gz