---
layout: post
title:  "Installation of ArchLinux ARM on an out-of-tree Amlogic device / 在不被官方支持的Amlogic设备上安装ArchLinux ARM"
date:   2022-11-08 21:10:07 +0800
categories: embedded
---

*除了这段话，中文始终在其对应英文的下一段落*

## The Live environment / 安装环境

You'll need a live environment to do the installation, like how we need an ArchLinux installation ISO or other distros for the task for installation of ArchLinux on x86-64.   
你需要一个安装环境来进行安装，就像我们在x86-64的主机上安装ArchLinux时需要ArchLinux的安装镜像或其他发行版

The main purpose is to run several tools including Pacman (ArchLinux's package manager) **natively** on the machine itself, and potentially write stuffs to eMMC depending on the target drive which is almost impossible if you're not operating on the box (for several SBCs including Odroid N2, this should be possible as they can function as a card reader for eMMC; but not for most set-up boxes so we won't cover this).  
需要这个安装环境的主要目的是在目标机器本身上运行包括Pacman（ArchLinux的包管理器）在内的几个工具，并且也可能根据安装的目标驱动器需要向eMMC直写，而直写不直接在盒子本身上操作的话是几乎不可能的（对于一些包括Odroid N2在内的开发板来说，这是可能的，因为它们可以当成eMMC的读卡器使用；不过大多数的机顶盒都不可以这么做，所以我们不在此讨论此情况）

A pre-built image for Armbian (e.g. [ophub's Armbian][ophub's Armbian] or other well-known distros are good enough to give you this live environment, but even CoreELEC/EmuELEC/OpenWrt should do the work (yeah). If you have a bootable ArchLinux ARM this will be easier (where you probably want to migrate/duplicate instead of fresh installation) as some easy-to-use installation tools included in [arch-install-scripts][arch-install-scripts] package are right there (``pacstrap``, ``genfstab``, etc), otherwise you might need to steel these tools from ArchLinux Package mirror or an existing installation e.g. x86-64.  
预构建的Armbian（比如 [ophub的Armbian][ophub's Armbian]）或者其他知名发行版的镜像是充当这个安装环境的绝佳选择，不过即使你用CoreELEC/EmuELEC/OpenWrt也不会太妨碍整个安装的流程。如果你本身已经有了一个能启动的ArchLinux ARM那就更好了（不过这种情况下你可能想要迁移/克隆而不是全新安装），因为有的在[arch-install-scripts][arch-install-scripts]里的易于使用的安装工具唾手可得（``pacstrap``, ``genfstab``等），不然的话你可能需要从ArchLinux的镜像站或者是一个现存的比如x86-64平台上的安装里借用这些工具。

I'll assume you can easily get this live environment up and not waste time wrting its steps here. **You can certainlly use an x86-64 host as the live environment, if you're just installing on a SDcard; but you probably have to do all of these on the box itself if you want to install to eMMC**    
我就假设你能简简单单地搭建起这个安装环境，不会浪费时间写怎么安装了。**如果只是要装到SD卡上的话，你确实可以用一个x86-84的宿主来充当这个安装环境；但只要你想安装到eMMC里，你基本就得在盒子本身上操作**

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
localhost   127.0.0.1
localhost   ::1
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
### Background / 背景
*If you know the bootloader machanism on Amlogic platform very well, you can jump to [setups][setups] to choose one of the setups. Otherwise better get yourself acknowledged for Amlogic's boot flow from this part  
如果你熟知Amlogic平台上的启动机制，你可以直接跳到[安装方法][setups]章节来选择一种安装方法。不然的话最好通过这一章节充分了解Amlogic的启动流程*

Several different bootloader/pre-boot environment combination can be chosen depending on the support status of your hardware in the upstream u-boot project due to the following boot flow:  
根据你的硬件在上游u-boot项目里的支持情况不同，几种不同的引导程序/启动前环境的组合可以选择，这是因为以下的启动流程：
```
 media-independent | load BL1 from ROM in SoC
    early boot     |            v
--------------------------------v-------------------------------
 media-dependent   | load BL2 from boot media
    early boot *1  |            v
   (eMMC->SD *2)   | load BL30 from boot media
                   |            v  
                   | load BL31 from boot media
                   |            v
                   | load BL33/u-boot from boot media
                   |            v
-------------------|------------v-----------------------------------
 1st stage u-boot*3| load u-boot | load kernel, fdt, prepare initrd
                   |      v                                  v
(2nd stage uboot)*4| load kernel, fdt, prepare initrd *5     v
                   |      v--^                               v
-------------------|------v----------------------------------v------
  kernel space     | mount initrd to load loadable modules
                   |                  v
-------------------|------------------v-----------------------------
  early userspace/ | mount rootfs
    ramfs          |     v
                   | run init
                   |     v
-------------------|-----v------------------------------------------
   userspace       |
```
1. Usually BL2, BL30, BL301, BL31 and BL33 (u-boot) are packed into a single signed, encrpted bootloader image, and will be flashed to the first several MiBs on the boot media. For stock Android images this will always be the first 4MiB  
通常，BL2, BL30, BL301, BL31 和 BL33(u-boot) 被打包成一个经过签名和加密的引导程序镜像，然后会被写入到启动媒介上最前面的几MiB。对于原厂的安卓镜像来说，总是最前面4MiB
2. The boot routine will try the whole BL2->BL30->BL31->BL33 process on the **same** boot media, then fall back to another if any step of the process has failed. The eMMC->SD order is just for the ease of read, in the real world Amlogic SoCs has other boot medias to prefer (check [boot flow on upstream u-boot's documentation][boot flow]), and failing at loading bootloader from eMMC will not neccessarily make the SoC redo the whole process from SD (some dead-locks might be entered due to bad BL2/BL30/BL31/BL33).  
引导流程会在**同一个**启动媒介上尝试整个BL2->BL30->BL31->BL33流程，然后如果任何一步失败后会回落到下一启动顺位。eMMC->SD的顺序在这里只是便于阅读，实际上Amlogic的SoC有若干不同的启动顺序组合(请查阅 [上游u-boot文档中的启动流程][boot flow])， 并且从eMMC加载引导程序失败也不一定就会让SoC从SD卡重走整个流程（因为有问题的BL2/BL30/BL31/BL33，整个加载流程可以能进入死锁）
3. The first stage u-boot **must** be the BL33 blob loaded earlier from bootloader image by BL31.  
第一阶段的u-boot**必需**是由BL31从引导程序镜像里加载的BL33
4. Second stage u-boot is only for the comprehension, it can be any ``u-boot.bin`` or ``u-boot.elf`` that you can build from either Amlogic's source or mainline u-boot, and can be entered directly by running ``go`` command in the earlier u-boot after loading it from a storage media. It must be exectuable and **not encrpyted like the the first stage bootloader, because we're in the BL33 world and there's no more BL31 to decrypt the BL33 like when entering first stage u-boot**. And it's totally possible to chain infinite number of u-boots by ``go``, we call all of them second stage u-boot  
写作第二阶段u-boot只是为了便于理解，它可以是任何的你可以从Amlogic或者主线u-boot源码构建的``u-boot.bin``或者``u-boot.elf``，并且可以直接由更早阶段的u-boot从存储媒介加载后通过``go``命令进入。它必需是可以执行的，并且**不能像第一阶段引导程序那样是加密的，因为我们已经进入到了BL33，不再像进入第一阶段u-boot时那样有BL31来解密BL33**。完全有可能通过``go``来链式启动无限多的u-boot，我们把所有这样的u-boot叫做第二阶段u-boot
5. Different from x86, the init ramdisk **must** be prepared (i.e. loaded to RAM) by the bootloader instread of the kernel on ARM, and it's the job of the same u-boot to load the kernel. Depending on the booting mechanism this can either be a normal initramfs (possible with syslinux config ``/extlinux/extlinux.conf`` style bootup in mainline u-boot) or legacy u-boot image with a type of initrd that can be created from a normal initramfs with ``mkimage``  
和x86不同，在ARM上，初始化的内存盘**必需**由引导程序而不是内核来准备（也就是加载到随机访问内存中），并且应当由加载内核的同一个u-boot去加载。根据启动机制的不同，这个内存盘镜像可以是普通的initramfs（用主线u-boot通过syslinux配置文件``/extlinux/extlinux.conf``启动情况下），或者是可以通过``mkimage``从普通的initramfs创建的类型为内存盘的传统的u-boot镜像

*the BL33 blob is loaded by earlier totally closed-
 source blobs, so BL33 is the earliest stage you can
 manipulate, unless you are a hardcode reverse-enginerring guy like me*  
*BL33代码是由更早的完全闭源的代码加载的，所以BL33是你能动手脚的最早的环节，除非你向我一样是一个热衷于逆向工程的人*

### Setups / 安装方法
*UART connection and backup is recommended before installing the bootloader    
在安装引导程序前，建议确保UART连接和备份*

You can find pre-built resources for bootloader (the whole BL2 to BL33/u-boot chain packed as a single image) and u-boot (plain u-boot a.k.a. BL33) in [ophub's repo][ophub bl] (called bootloader and overload accordingly in the repo)  
你可以在[ophub的仓库][ophub bl]里找到预构建的引导程序（整个BL2到BL33/u-boot引导链的打包镜像）和u-boot（普通的u-boot也就是BL33）（在仓库里分别叫做bootloader和overload）

The following bootloader setups can be used depending on what kind of bootloader/u-boot you could get for your device:  
可以使用以下的引导程序配置：  
#### Mainline as bootloader / 主线作为引导程序
Write signed bootloader image with mainline u-boot on eMMC/SD, create a fat/ext4 partition as rootfs on eMMC/SD, containing syslinux config or uboot script, plug kernel, initrd and fdt to startup, if it's ext4 it could be the rootfs itself  
把含有主线u-boot的引导程序镜像写到eMMC或者SD上，在eMMC或者SD上创建一个fat/ext4分区来储存syslinux配置或者是uboot脚本，加上内核，内存盘和设备树来启动，如果是ext4的话，这个分区可以是根文件系统本身      
Steps to install a bootloader with mainline u-boot as BL33 onto eMMC, when ``mmcblk2`` is the block device corresponding to eMMC and ``u-boot.bin.sd.bin`` is the bootloader image:  
假设``mmcblk2``是eMMC对应的块设备，``u-boot.bin.sd.bin``是引导程序镜像，要把一个里面以主线u-boot作为BL33的引导程序镜像写入eMMC的步骤：  
1. *Backup of the on-eMMC bootloader is optional, you can skip this step if you're confident that nothing will break  
 备份eMMC上的引导程序是可选的，如果你足够自信不会有东西崩掉，你可以跳过这一步*
    ```
    dd if=/dev/mmcblk2 of=bootloader.img.backup bs=1M count=4
    ```
2. To write the image to eMMC, and avoid overwriting byte 445-512 in which the MBR partition table is stored  
    把镜像写入到eMMC，并避免覆盖第445至512字节，这是储存MBR的分区表的位置
    ```
    dd if=u-boot.bin.sd of=/dev/mmcblk2 conv=fsync,notrunc bs=1 count=444
    dd if=u-boot.bin.sd of=/dev/mmcblk2 conv=fsync,notrunc bs=512 skip=1 seek=1
    ```
*Tested device: BPi M5, as it has upstream u-boot support, and a [documentation][build u-boot] to follow to build and sign the u-boot (It's a C4 clone with minor pinout differences so the C4 doc can be followed for the most)  
测试过的设备：BPI M5，由上游u-boot的支持，并且有[文档][build u-boot]可以参照来构建和签名u-boot（M5本身是个C4换皮，有少许引脚差别，所以C4的文档一样适用）*  
*Tested device: HK1 Box, as someone has compiled a mainline u-boot for it packed as bootloader image that's included in [amlogic-s9xxx-armbian][amlogic-s9xxx-armbian]. With tools like [gxlimg][gxlimg] you can also sign a mainline u-boot for u200 or similar devices with minor tweaks, and pack it at your bootloader image, it's pretty easy to port. There're also projects like flippy's [u-boot][unifreq's u-boot] and [amlogic-boot-fip][unifreq's amlogic-boot-fip] with already ported sources to refer to  
   测试过的设备：HK1 Box，[amlogic-s9xxx-armbian][amlogic-s9xxx-armbian]项目中有给这个设备打包好的包含主线u-boot的引导程序镜像。通过像是[gxlimg][gxlimg]这样的工具你也能给u200或是类似设备的主线u-boot签名并打包成你的引导程序镜像，移植起来很简单。还有像是flippy的[u-boot][unifreq's u-boot]和[amlogic-boot-fip][unifreq's amlogic-boot-fip]这样由已经移植过的源码的项目可以参照*  

#### Mainline/dirty as 2nd stage / 主线/移植作为二阶段
Keep stock signed encrpted bootloader image with stock u-boot on eMMC as the 1st stage u-boot, load a mainline/dirty (newer, purposedly built Amlogic) u-boot from a FAT partition on eMMC/SD/USB, and then a fat or ext4 on eMMC/SD containing syslinux config or bootscript, plus kernel, initrd and fdt to startup. If it's ext4 then it can be the rootfs itself, if it's fat it can be the exact same partition uboot is stored in.  
在eMMC上保留原厂的签名过且加密过的引导程序镜像和里面的原厂u-boot作为第一阶段u-boot，从eMMC/SD/USB上的FAT分区加载主线或者是脏的（更新的，特别构建的Amlogic）u-boot，然后以一个eMMC/SD上的一个fat或者是ext4分区加载syslinux配置或是引导脚本，加上内核、初始化内存盘和设备树。如果这个分区是ext4的话，那它可以是根文件系统本身；如果这个分区是fat的话，那它可以就是uboot被储存在的同一个分区
1. (linux) To mount the u-boot partition you created earlier  
挂载你所创建的有FAT文件系统的uboot分区并把你的主线u-boot储存在其中  
    *when uboot is in its dedicated partition  
    当uboot放在它专属的分区里时*
    ```
    mount /dev/uboot_partition /mnt/boot/uboot
    ```
    *when uboot is side-by-side when kernel, initramfs and fdt  
    当uboot和内核，初始化内存盘和设备树放在一起时*
    ```
    mount /dev/boot_partition /mnt/boot
    ```
2. (linux) To store the mainline u-boot in that partition  
    *when uboot is in its dedicated partition  
    当uboot放在它专属的分区里时*
    ```
    cp u-boot.bin /mnt/boot/uboot/mainline
    ```
    *when uboot is side-by-side when kernel, initramfs and fdt  
    当uboot和内核，初始化内存盘和设备树放在一起时*
    ```
    cp u-boot.bin /mnt/boot/mainline
    ```
3. (u-boot) Make sure the stock u-boot will load the mainline/dirty u-boot when booting, e.g. when ``/dev/uboot_partition`` is the 1st partition on eMMC (eMMC is usually mmc1, SD is usually mmc0), the following commands should be executed by the stock u-boot  
确保原厂的u-boot会在启动时加载主线/脏的u-boot，比如，当``/dev/uboot_partition``是eMMC上的第一个分区时(eMMC一般是mmc1，SD一般是mmc0)，下面的命令应该被原厂u-boot执行
    ```
    fatload mmc 1:1 0x1000000 mainline
    go 0x1000000
    ```
    *Note the syntax of ``fatload`` is ``fatload [devtype] [devnum](:[partnum]) [hex address] [filename]``, where ``devtype`` can be one of ``mmc``, ``usb`` (**Additional command ``usb start`` should be executed before a ``usb`` type device can be read**), and ``devnum`` starts at 0, ``partnum`` starts at 1 (can be omitted when it's 1).    
    注意``fatload``的语法是``fatload [设备类型] [设备号](:[分区号]) [16进制地址] [文件名]``，其中``设备类型``可以是``mmc``, ``usb``（**要从``usb``类型的设备读取，一条额外的命令``usb start``应该在此之前被执行**），``设备号``从0开始，``分区号``从1开始（当为1时可以被省略）  
    It's recommended to **confirm the exact partition identifier with ``fatls [devtype] [devnum](:[partnum])`` in u-boot first** to make sure the ``[devtype] [devnum](:[partnum])`` identifier points to the exact partition where you store your mainline u-boot  
    建议**先在u-boot下通过``fatls [设备类型] [设备号](:[分区号])``来确认**标识符``[设备类型] [设备号](:[分区号])``指向的分区确确实实是你放主线u-boot的分区*  
    This bootcmd can be set in the following ways:  
    这条启动命令可以通过以下方式设置：
    1. To setting bootcmd in u-boot itself (or with more complicated logic, you can use other variable as essentially functions with ``run variable``):  
    在u-boot里设置``bootcmd``来实现（或者用更复杂的逻辑，你可以把其他变量通过``run 变量名``基本当成函数使用）
        ```
        defenv
        setenv bootcmd 'fatload mmc 1:1 0x1000000 mainline; go 0x1000000'
        saveenv
        reboot
        ```
        *If you want to fallback to Android booting, change the line ``setenv bootcmd`` to this instead:  
        如果你想要能回落到安卓启动的话，把``setenv bootcmd``这行换成这样*
        ```
        setenv bootcmd 'if fatload mmc 1:1 0x1000000 mainline; then go 0x1000000; else run storeboot; fi'
        ```
    2. To use ``fw_setenv`` in Linux, remember to set ``/etc/fw_env.config`` first with a line like this:  
    通过在Linux下使用``fw_setenv``，记得先在``/etc/fw_env.config``下配置这样的一行：
        ```
        /dev/your_emmc_drive 0x7400000 0x10000
        ```
        *The number ``0x7400000`` and ``0x10000`` here mean offset at 116MiB and size 64KiB, you'll need to use my partition tool [ampart][ampart] first to check the actual offset of EPT partition ``env`` first. ampart can be built with a simple ``make`` with only dependency on ``zlib``, and the EPT can be checked with a simple ``ampart /dev/your_emmc_drive``  
        数字``0x7400000``和``0x10000``在这里分别是偏差116MiB和大小64KiB，你需要用我的工具[ampart][ampart]先来检查EPT分区``env``的偏差。ampart可以通过一条简单的``make``来构建，仅有的依赖是``zlib``，而EPT分区表也可以通过一条简单的``ampart /dev/your_emmc_drive``来检查*  

        Then actually use ``fw_setenv``:  
        然后再使用``fw_setenv``
        ```
        fw_setenv bootcmd 'fatload mmc 1:1 0x1000000 mainline; go 0x1000000'
        ```
        *If you want to fallback to Android booting, use a bootcmd like this instead:  
        如果你想要能回落到安卓启动的话，用一条像这样的bootcmd*
        ```
        fw_setenv bootcmd 'if fatload mmc 1:1 0x1000000 mainline; then go 0x1000000; else run storeboot; fi'
        ```
    3. To enter update mode (holding reset or ``reboot update`` with root in Android) with a usb drive/SD card attached with an ``aml_autoscript`` generated with ``mkimage`` from a plain text file with the above content in way 1 on its first partition that's formated with a FAT fs. Which could be generated with the following command:  
    也可以通过在插着第一个分区格式化为FAT且包含由第一种方式中的内容的纯文本经由``mkimage``生成的``aml_autoscript``的情况下进入升级模式（按住重置键或者在安卓下以root权限输入``reboot update``），这个``aml_autoscript``可以通过下面这条命令来生成
        ```
        mkimage -A arm64 -O linux -T script -C none -d /path/to/plain/text/script/to/the/above/content /path/to/the/generated/aml_autoscript
        ```
        *If opening with a text editor, the result file might seem glitched, don't worry, a u-boot script will containing extra header  
        如果用一个文本编辑器打开结果的文件，看起来可能像是乱码，不用担心，u-boot脚本包含有额外的头部信息*
        ```
        'Vet2�ci�k`��kXdefenv
        setenv bootcmd 'fatload mmc 1:1 0x1000000 mainline; go 0x1000000'
        saveenv
        reboot
        ```
        
*Tested device: BesTV R3300L, as it's basically a p211 clone, which is then a shrinked p212, which is then supported by upstream u-boot. Only minor DTS adjustment is needed to port the p212 mainline u-boot to it. Yet then BL33 is nearly impossible to sign due to limited p211 sources (address tests will fail, don't even try that)  
测试过的设备：百视通R3300L，基本上就是一个p211的换皮，而p211又是个缩水过的p212，而p212是被上游u-boot支持的。只需要做简单的DTS调整就能把p212的主线内核移植上来。不过因为有限的p211源码，BL33几乎不可能被签名（地址测试会失败，别想试了）* 

#### Stock bootloader all the way / 自始至终原厂引导程序
If you don't want to break anything, you can keep the stock bootloader and does not load any u-boot, and just rely on the stock u-boot to load the kernel. The stock u-boot then needs to either run ``booti`` itself, or load other scripts to do the work  
如果你不想整坏任何东西，你可以保留原厂的引导程序，并且不加载任何的u-boot，仅依靠原厂的u-boot来加载内核。这样的话原厂的u-boot就需要要么自己跑``booti``，要么加载其他脚本来调用``booti``  
The basic idea is to load kernel, initrd and fdt from a FAT fs, the most essential commands include these, if, for example, you are storing these in the first partition on eMMC (see above and below for ``fatload`` syntax):  
基本的思路是从一个FAT分区来加载内核，初始化内存盘和设备树，假如你把这些存放在eMMC上的第一个分区的话，最重要的命令包含这些（请从上下其他部分看``fatload``的语法）
```
fatload mmc 1:1 0x1000000 dtbs/linux-aarch64-flippy/amlogic/meson-sm1-hk1box-vontar-x3.dtb
fatload mmc 1:1 0x1080000 vmlinuz-linux-aarch64-flippy
fatload mmc 1:1 0x3080000 initramfs-linux-aarch64-flippy
setenv bootargs root=UUID=14a0b881-79c2-47b7-ae2d-94b0d09b7451 rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes
booti 0x1080000 0x3080000 0x1000000
```
*Addresses ``0x1000000``=16MiB, ``0x1080000``=16.5MiB, ``0x3080000``=48.5MiB, this means there's at most 512KiB for FDT and 32MiB for the kernel, depending on your actual setup these values might need to be increased. These values might be pre-defined as variables in your u-boot, e.g. ``loadaddr``, ``kernel_addr_r``, etc, but I made them as plain values here to help you understand the whole loading process  
地址``0x1000000``=16MiB, ``0x1080000``=16.5MiB, ``0x3080000``=48.5MiB，这意味着FDT最大可以512KiB，内核最大可以32MiB，根据你的实际配置这些值可能要增加。这些值可能在你的u-boot里已经作为变量预先定义了，比如``loadaddr``, ``kernel_addr_r``等，不过我这里把它们作为直白的数字写明，以帮助你理解启动流程*

Or if you prefer to store the variables in an easy-to-edit textfile, e.g. ``uEnv.txt``  
如果如果你更喜欢把这些变量储存在一个易于编辑的文本文件，比如说``uEnv.txt``里
```
LINUX=vmlinuz-linux-aarch64-flippy
INITRD=initramfs-linux-aarch64-flippy
FDT=dtbs/linux-aarch64-flippy/amlogic/meson-sm1-hk1box-vontar-x3.dtb
APPEND=root=UUID=14a0b881-79c2-47b7-ae2d-94b0d09b7451 rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes
```
Then the commands will be much tidier  
那么命令就会更简洁
```
fatload mmc 1:1 0x1000000 uEnv.txt
env import -t 0x1000000
fatload mmc 1:1 0x1000000 ${FDT}
fatload mmc 1:1 0x1080000 ${LINUX}
fatload mmc 1:1 0x3080000 ${INITRD}
setenv bootargs ${APPEND}
booti 0x1080000 0x3080000 0x1000000
```
If you can't modify u-boot envs directly, please refer to the latter part to learn to to write these commands as a script and use ``aml_autoscript`` to enable it  
如果你不能直接编辑u-boot环境，请参考之后的部分来了解如何把这些命令写成脚本并使用``aml_autoscript``来启用它


#### Others / 其他方案
There's also other choices, you can create your bootup machanism freely as long as the boot flow won't break  
也有其他的选择，你可以自由地创建你的启动机制，只要整个启动流程不要崩掉

## Booting configuration / 启动配置
After you got the bootloader set up, now your shinny new u-boot needs configuration to actually boot your system. Depending on the last stage u-boot which will load the kernel, you have different configurations to use:  
既然你已经搭建好了引导程序，现在你崭新的u-boot就需要配置文件来真正地启动你的系统了。根据最后一阶段的要加载内核的u-boot，你有以下不同的配置来选择

|style|file|mainline|dirty|stock|initramfs|  
|-|-|-|-|-|-|
|syslinux (extlinux)|``/extlinux/extlinux.conf`` ``/boot/extlinux/extlinux.conf``|yes|depends|no|standard legacy
|script|``/boot.scr`` ``/boot.scr.uimg`` ``/boot/boot.scr`` ``/boot/boot.scr.uimg``|yes|yes|yes|legacy
|aml_autoscript*|``/aml_autoscript`` ``/aml_autoscript.zip``|no|depends|yes|legacy

### syslinux (extlinux)
Mainline u-boot, by default, run ``scan_dev_for_extlinux`` first for each potentially bootable partition, scanning for ``/extlinux/extlinux.conf`` which optional prefix ``/boot``, which then use u-boot command ``sysboot`` to boot, the configuration should be stored as ``/extlinux/extlinux.conf`` or ``/boot/extlinux/extlinux.conf`` relative to the root of the fs. The configuration has the same syntax as documented in [syslinux's documentation][syslinux config], and is pretty straight-forward to write. An example configuration I use for HK1Box:  
默认情况下，主线u-boot会首先在每个可能可以启动的分区上运行``scan_dev_for_extlinux``，扫描可能有前缀``/boot``的``/extlinux/extlinux.conf``，然后再调用u-boot命令``sysboot``来启动，配置文件应当被储存为``/extlinux/extlinux.conf``或是``/boot/extlinux/extlinux.conf``，相对于文件系统的根目录。配置文件的语法和在[syslinux的文档][syslinux config]里记录的一样，写起来很简单直白。一个我在HK1Box上使用的配置:  
```
LABEL   Arch Linux
LINUX   /boot/vmlinuz-linux-aarch64-flippy
INITRD  /boot/initramfs-linux-aarch64-flippy.img
FDT     /boot/dtbs/linux-aarch64-flippy/amlogic/meson-sm1-hk1box-vontar-x3.dtb
APPEND  root=UUID=14a0b881-79c2-47b7-ae2d-94b0d09b7451 rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes
```
*(The configuration, kernel, initramfs and fdt are all stored inside a subfolder ``boot`` inside the rootfs itself. If they are seperated and stored directly inside another partition, the following should be used instead:)  
（配置文件，内核，初始化内存盘和设备树都储存在root本身下面的一个子目录``boot``里，如果它们是分开储存并且直接在另一个分区里，应该使用下面的例子：）*
```
LABEL   Arch Linux
LINUX   /vmlinuz-linux-aarch64-flippy
INITRD  /initramfs-linux-aarch64-flippy.img
FDT     /dtbs/linux-aarch64-flippy/amlogic/meson-sm1-hk1box-vontar-x3.dtb
APPEND  root=UUID=14a0b881-79c2-47b7-ae2d-94b0d09b7451 rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes
```
*Since multiple labels could exist in a single syslinux config, you could indent the lines below the ``LABEL`` line for style, but I like to keep them all not indented when there's only one boot label  
因为单一的syslinux配置里可以存在多个标签，为了风格化，你可以把``LABEL``下面的行缩进。不过我喜欢在只有一个启动标签的时候不缩进*  
As you may find from the configuration, a standard initramfs can be directly used as ``INITRD``, this has also the benefit of easily update and management (since there's only one file for config and one file for initramfs, in which the initramfs is covered by the mkinitcpio Pacman hook), no endless u-boot image packcing B$ is involved  
你能从配置里发现，一个普通的初始化内存盘可以直接作为``INITRD``来使用，这种配置方式也有易于升级和管理的好处（因为只有一个文件作为配置，一个文件作为初始化内存盘，其中初始化内存盘由mkinitcpio的一个Pacman钩子负责），没有无穷无尽的u-boot镜像打包

### script / 脚本
Regardless of what u-boot you are using, a script can be loaded from any fs the u-boot support and executed, as long as it's stored in u-boot script format (or plain text with a dedicated header, if you're using an Odroid or similar u-boot)  
不管你是用的是什么u-boot，都能从这个u-boot支持的文件系统中加载脚本并执行，只要这个脚本以u-boot脚本的格式打包（或者是纯文本，不过有特定的开头，如果你用的是Odroid或者类似的u-boot的话）

Mainline u-boot, by default, will run ``scan_dev_for_scripts`` after scanning for syslinux for each partition, and potential script targets include ``boot.scr.uimg`` and ``boot.scr``, with optional prefix ``/boot``. This might also be supported in the dirty/stock u-boot, and can be implemented with easy setenv for stock u-boot (It's recommended to set them with the method mentionded in the [aml_autoscript](#aml_autoscript))  
主线内核在默认情况下会对每个分区在扫描完syslinux后扫描脚本，扫描有可选前缀``/boot``的脚本``boot.scr.uimg``和``boot.scr``。这也能在原厂或是粗暴移植的u-boot中得到支持，并且能通过简单的setenv在原厂u-boot上实现（建议通过下面[aml_autoscript](#aml_autoscript)中的方式实现）

An example boot script that's used in the [amlogic-s9xxx-armbian][amlogic-s9xxx-armbian] project:  
例如[amlogic-s9xxx-armbian][amlogic-s9xxx-armbian]项目中使用的启动脚本
```
echo "Start AMLOGIC mainline U-boot"
if printenv bootfromsd; then exit; fi;
setenv loadaddr "0x44000000"
setenv l_mmc "0 1 2 3"
for devtype in "usb mmc" ; do
  if test "${devtype}" = "mmc"; then
    setenv l_mmc "1"
  fi 
  for devnum in ${l_mmc} ; do
    if test -e ${devtype} ${devnum} uEnv.txt; then
      load ${devtype} ${devnum} ${loadaddr} uEnv.txt
      env import -t ${loadaddr} ${filesize}
      setenv bootargs ${APPEND}
      if printenv mac; then
        setenv bootargs ${bootargs} mac=${mac}
      elif printenv eth_mac; then
        setenv bootargs ${bootargs} mac=${eth_mac}
      elif printenv ethaddr; then
        setenv bootargs ${bootargs} mac=${ethaddr}
      fi
      if load ${devtype} ${devnum} ${kernel_addr_r} ${LINUX}; then
        if load ${devtype} ${devnum} ${ramdisk_addr_r} ${INITRD}; then
          if load ${devtype} ${devnum} ${fdt_addr_r} ${FDT}; then
            fdt addr ${fdt_addr_r}
            booti ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}
          fi
        fi
      fi
    fi
  done
done
# Recompile with:
# mkimage -C none -A arm -T script -d /boot/boot.cmd /boot/boot.scr
```
As you can see the file has a syntax similar to shell scripting, since u-boot uses hush shell and some common shell syntaxes are ported, but you won't find your familiar shell tools and couldn't use fancy stuffs like piping.  
如同所见，这个文件的语法和shell脚本很相似，因为u-boot是用的是hush shell，然后一些常见的shell语法被移植了，不过你找不到熟悉的shell工具，并且也不能整一些诸如管道之类的花活

Different from syslinux config, you have do all kinds of different things in a single script, as long as it's supported by u-boot itself. So a boot target in u-boot can then boot multiple targets in script thanks to some basic loops.  
和syslinux配置不同，你可以在一个脚本里做各种各样的事情，只要u-boot本身能支持。所以u-boot里的单一启动目标可以启动多个启动目标，多亏了一些基本的循环逻辑

The plain-text boot script can then be compiled into a u-boot script with the following command  
纯文本的启动脚本可以通过下面的命令来编译成u-boot脚本
```
mkimage -C none -A arm -T script -d /path/to/plain/text/script /path/to/uboot/script
```

### aml_autoscript
This is a special case for stock u-boot that's basically the same as the script above, the only difference is that it'll be automatically executed by the stock u-boot if the device is booted in update mode (either through Android or u-boot ``reboot update`` or a cold boot with reset button held down). Due to this nature, we can use this script as an entry-point that'll prepare the u-boot env for consecutive boots   
这是一个原厂u-boot的特别情况，基本和前述的脚本原理一样，唯一的区别是这个脚本会自动被原厂的u-boot在升级模式（通过安卓或u-boot下``reboot update``或者按住重置按钮来冷启动）下自动执行。因为这一特点，我们可以把这个脚本当作入口脚本，来为之后的启动设置u-boot环境

These are the sources for ``aml_autoscript``, ``s905_autoscript`` and ``emmc_autoscript`` from project [amlogic-s9xxx-armbian][amlogic-s9xxx-armbian]. As you can see the first script ``aml_autoscript`` sets up the environment variables ``bootcmd``, ``start_autoscript``, ``start_emmc_autoscript``, ``start_mmc_autoscript`` and ``start_usb_autoscript``. In later boots the new ``bootcmd`` will be effective and function as the entry point for the mmc->usb->emmc->storeboot priority, and the scripts ``s905_autoscript`` and ``emmc_autoscript`` will be used accordingly:  
下面是[amlogic-s9xxx-armbian][amlogic-s9xxx-armbian]项目中``aml_autoscript``, ``s905_autoscript``, ``emmc_autoscript``的脚本。如你所见第一个脚本``aml_autoscript``设置了环境变量``bootcmd``, ``start_autoscript``, ``start_emmc_autoscript``, ``start_mmc_autoscript``和``start_usb_autoscript``。之后的启动中新的``bootcmd``就会生效，并充当mmc->usb->emmc->storeboot这一引导顺序的入口点，并且脚本``s905_autoscript``和``emmc_autoscript``会分别被使用

*aml_autoscript*
```
if printenv bootfromsd; then exit; else setenv ab 0; fi;
setenv bootcmd 'run start_autoscript; run storeboot'
setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then autoscr 1020000; fi;'
setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi;'
setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then autoscr 1020000; fi; done'
setenv upgrade_step 2
saveenv
sleep 1
reboot
```
*s905_autoscript*
```
echo "start amlogic old u-boot"
if fatload mmc 0 ${loadaddr} boot_android; then if test ${ab} = 0; then setenv ab 1; saveenv; exit; else setenv ab 0; saveenv; fi; fi;
if fatload usb 0 ${loadaddr} boot_android; then if test ${ab} = 0; then setenv ab 1; saveenv; exit; else setenv ab 0; saveenv; fi; fi;
if fatload mmc 0 0x1000000 u-boot.ext; then go 0x1000000; fi;
if fatload usb 0 0x1000000 u-boot.ext; then go 0x1000000; fi;
setenv kernel_addr_r 0x11000000
setenv ramdisk_addr_r 0x13000000
setenv fdt_addr_r 0x1000000
setenv l_mmc "0"
for devtype in "mmc usb" ; do if test "${devtype}" = "usb"; then echo "start test usb"; setenv l_mmc "0 1 2 3"; fi; for devnum in ${l_mmc} ; do if test -e ${devtype} ${devnum} uEnv.txt; then fatload ${devtype} ${devnum} ${loadaddr} uEnv.txt; env import -t ${loadaddr} ${filesize}; setenv bootargs ${APPEND}; if printenv mac; then setenv bootargs ${bootargs} mac=${mac}; elif printenv eth_mac; then setenv bootargs ${bootargs} mac=${eth_mac}; fi; if fatload ${devtype} ${devnum} ${kernel_addr_r} ${LINUX}; then if fatload ${devtype} ${devnum} ${ramdisk_addr_r} ${INITRD}; then if fatload ${devtype} ${devnum} ${fdt_addr_r} ${FDT}; then fdt addr ${fdt_addr_r}; booti ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}; fi; fi; fi; fi; done; done;
```

*emmc_autoscript*
```
if fatload mmc 1 0x1000000 u-boot.emmc; then go 0x1000000; fi;
setenv dtb_addr 0x1000000
setenv env_addr 0x1040000
setenv kernel_addr 0x11000000
setenv initrd_addr 0x13000000
setenv boot_start booti ${kernel_addr} ${initrd_addr} ${dtb_addr}
setenv addmac 'if printenv mac; then setenv bootargs ${bootargs} mac=${mac}; elif printenv eth_mac; then setenv bootargs ${bootargs} mac=${eth_mac}; elif printenv ethaddr; then setenv bootargs ${bootargs} mac=${ethaddr}; fi'
if fatload mmc 1 ${env_addr} uEnv.txt && env import -t ${env_addr} ${filesize}; setenv bootargs ${APPEND}; then if fatload mmc 1 ${kernel_addr} ${LINUX}; then if fatload mmc 1 ${initrd_addr} ${INITRD}; then if fatload mmc 1 ${dtb_addr} ${FDT}; then run addmac; run boot_start; fi; fi; fi; fi;
```

If you are happy with the boot priority, you can steal the pre-built scripts from that project, or build these with the following command  
如果你对这种启动顺序满意的话，你可以从那个项目中借用预构建的脚本，也可以用下面这条命令把上面的内容打包成脚本镜像
```
mkimage -A arm64 -O linux -T script -C none -d /path/to/plain/text/script/to/the/above/content /path/to/the/generated/aml_autoscript
```
If all you want is a simple no-wait boot for e.g. mmc1(eMMC), on which mainline u-boot is stored, the following content as ``aml_autoscript`` is fast and simple, which is what I prefer  
如果你想要的是毫无等待地从比如说mmc1(eMMC)启动，上面储存着主线u-boot，下面的内容作为``aml_autoscript``又快又简单，我更喜欢
```
defenv
setenv bootcmd 'fatload mmc 1:1 0x1000000 mainline; go 0x1000000'
saveenv
reboot
```
or like the following, where the stock u-boot loads the kernel by itself  
或者像下面这样，原厂的u-boot自己来加载内核
```
defenv
setenv bootcmd 'fatload mmc 1:1 0x1000000 uEnv.txt; env import -t 0x1000000; fatload mmc 1:1 0x1000000 ${FDT}; fatload mmc 1:1 0x1080000 ${LINUX}; fatload mmc 1:1 0x3080000 ${INITRD}; setenv bootargs ${APPEND}; booti 0x1080000 0x3080000 0x1000000'
saveenv
reboot
```
or like the following, imitated the mainline behaviour to load ``boot.scr`` and execute it (``source`` must be supported (usually in mainline), otherwise use ``autoscr`` instead (usually in stock))  
或者像下面这样，模仿主线的行为，加载``boot.scr``并执行（``source``命令必须被支持（主线一般都是），否则请用``autoscr``（原厂一般都是））
```
defenv
setenv bootcmd 'fatload mmc 1:1 0x1000000 boot.scr; source 0x1000000'
saveenv
reboot
```

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
A big thing to remember is that the first few MiBs on the eMMC on Amlogic devices are usually used to store bootloader (a combined image of BL2, BL30, BL301, BL31 and BL33/u-boot), and it's usually 4MiB. Which overlays with the MBR partition table. You'll need to backup it before and restore it after the partitioning:  
Amlogic设备上的eMMC上最初的几个MiB通常是储存引导程序镜像的（BL2, BL3, BL30, BL31和BL33/u-boot的整合镜像），通常是4MiB。这一部分和MBR分区表重叠。在分区前后你需要备份和还原引导程序镜像

To backup the bootloader:  
备份引导程序镜像：
```
dd if=/dev/mmcblk2 of=bootloader.img bs=1M count=4
```
To restore the bootloader:  
还原引导程序镜像:
```
dd if=bootloader.img of=/dev/mmcblk2 conv=fsync,notrunc bs=1 count=444
dd if=bootloader.img of=/dev/mmcblk2 conv=fsync,notrunc bs=512 skip=1 seek=1
```
Take extra care if you're partitioning with GPT as the numbers here are only for MBR, but I don't see the point to have a GPT partition table on such tiny drive.  
如果你以GPT进行分区的话，要额外注意，这里的数字只是给MBR分区表用的，不过对于这么小的驱动器来说，我不觉得GPT分区表有什么用。

And you shouldn't create partition to overlap with these 4MiB (avoid first 8192 sectors if you're using ``fdisk``), there's also other parts you shouldn't create partitions, continue reading to learn how to avoid and even shrink them  
并且你不应该创建覆盖这4MiB的分区（如果你用``fdisk``来分区的话，避开最开始的8192个扇区），同时也有其他你不应该创建分区的部分，继续阅读以了解怎么避开乃至缩小它们

If you're using a **stock Amlogic uboot** as the first stage u-boot in the bootloader (which will be the case unless you write a signed bootloader image with mainline u-boot onto the start of the eMMC, or to the SD and there's no bootloader on eMMC and you're using a SBC), you need to watch for Amlogic's proprietary eMMC partition table (EPT). The table and the creation logic behind it is why installation scripts on the market dare not touch the first 1GiB on the eMMC. The following is the EPT creation logic:  
如果你使用**Amlogic原厂的u-boot**作为引导程序镜像里面的第一阶段u-boot（除非你把里面有主线u-boot的引导程序镜像写到eMMC的开头，或者是SD卡的开头且用的是开发板，不然多数情况为此）。你需要注意Amlogic专有的eMMC分区表（EPT）。这个分区表和创建它的逻辑也就是为什么市面上的安装脚本不敢碰eMMC上最开始的1GiB，下面是它的创建逻辑。

1. The Amlogic u-boot will create several in-memory partition tables in the EPT format:  
Amlogic的u-boot会在内存里创建几个EPT分区表
   1. one of them is a template EPT with bootloader at 0-4M, reserved at 36-100M, cache at 108-108M and env at 116M-124M;  
   其中一个是模板EPT，bootloader区在0-4M，reserverd区在36-100M，cache区在108-108M，env区在116-124M
   2. another one is a virtual table that contains ddr, dtb and other stuffs that's usually stored inside reserved, for the ease of lookup;  
   另一个是一个包括ddr, dtb和其他通常储存在reserved区内的东西的虚拟分区表，为了便于查询
2. It reads the partitions node in DTB, then populates every partition in the partitions node into the emplate table mentioned earlier, each partition has a 8M gap before it, and the partition that has a size ``0xFFFFFFFFFFFFFFFF`` (u64, same hex as i64 -1) will occupy all of the remaining space  
u-boot会读取DTB里面的partitions节点，然后把partitions节点里的所有分区加载到前面的模板分区表里，每一个分区前面都会有8M的间隙，并且大小为``0xFFFFFFFFFFFFFFFF``（64位无符号整数，和64位有符号整数-1的十六进制相同）的分区会占用所有的剩余空间
3. If there's one such partition table stored in the reserved partition, compare these two, and always update the in-reserved one using the DTB one, unless the DTB one can't be created, e.g. there's no partitions node in DTB, and in that case just use the in-reserved one.  
如果reserved分区里面有这样一个分区表，对比两者，并且永远以从DTB构建的分区表来更新reserved分区内的分区表，除非DTB的那个不能被构建，比如说DTB里面没有partitions节点，那种情况下只用reserved分区内的那个
4. If there is MBR or GPT partition table on eMMC, the above table will not be used in the boot logic (commands including ``fatload``, ``extNload``, ``load``, etc), yet **still used** for looking up for ``misc``, ``logo``, etc partitions   
如果eMMC上有MBR或者GPT分区表，前述的分区表不会在启动逻辑中使用（``fatload``, ``extNload``, ``load``等这些命令），但**仍然会**用于``misc``, ``logo``等这些分区

What makes this a mess is that most vendors will create a cache partition either 512MiB or 1GiB, and due to the logic of step2 this will shift env to either ~600MiB or ~1.1GiB, and oh you should never create your partition that covers this naughty env partition.  
让这变得一团乱麻的原因是大多数制造商会创建一个512MiB或者是1GiB的cache区，因为第二步的逻辑这会把env区横移到大约600MiB或者大约1.1GiB的位置，然后哦，你永远不能在这个调皮的env分区的位置上面创建你的分区

Thankfully I've written a partition tool [ampart][ampart] for this partition format. It can be easily built on x86-64 or aarch64 with a easy ``make``, and can easily read and edit both the in-reserved EPT and the partitions node in DTB.  
幸亏我给这种分区格式写了一个分区工具[ampart][ampart]。ampart可以很简单地在x86-64或者是aarch64架构上通过简单的``make``来构建，并且能简单地读取和编辑reserved区里的EPT和DTB里的partitions节点

On Arch-derived distros you can install ampart through AUR package [ampart-git][ampart-git]:  
在Arch衍生的发行版上你可以通过AUR包[ampart-git][ampart-git]来安装ampart:
```
git clone https://aur.archlinux.org/ampart-git.git
cd ampart-git
makepkg -si
```
On Debian-derived distros (e.g. Ubuntu, Armbian) you can clone the [ampart repo][ampart] and build with a simple ``make``(remember to install built dependencies ``git``, ``build-essential`` and ``zlib1g-dev`` first):  
在Debian衍生的发行版（比如Ubuntu，Armbian）上你可以克隆[ampart仓库][ampart]并通过简单的``make``来构建（记得通过``apt``安装构建依赖``git``,``build-essential``和``zlib1g-dev``）：
```
git clone https://github.com/7Ji/ampart.git
cd ampart
make
```
A simple call with only the eMMC drive as argument will give you the EPT and partitions node in DTB:  
简单地把eMMC驱动器作为从参数来调用，你就能看到EPT和DTB里面的partitions节点：
```
ampart /dev/your_eMMC_drive
```
*partitions node in DTB:  
DTB里的分区：*
```
DTS report partitions: 17 partitions in the DTB:
=======================================================
ID| name            |            size|(   human)| masks
-------------------------------------------------------
 0: logo                       800000 (   8.00M)      1
 1: recovery                  1800000 (  24.00M)      1
 2: misc                       800000 (   8.00M)      1
 3: dtbo                       800000 (   8.00M)      1
 4: cri_data                   800000 (   8.00M)      2
 5: param                     1000000 (  16.00M)      2
 6: boot                      1000000 (  16.00M)      1
 7: rsv                       1000000 (  16.00M)      1
 8: metadata                  1000000 (  16.00M)      1
 9: vbmeta                     200000 (   2.00M)      1
10: tee                       2000000 (  32.00M)      1
11: vendor                   14000000 ( 320.00M)      1
12: odm                       8000000 ( 128.00M)      1
13: system                   74000000 (   1.81G)      1
14: product                   8000000 ( 128.00M)      1
15: cache                    46000000 (   1.09G)      2
16: data                              (AUTOFILL)      4
=======================================================
```
EPT:
```
EPT report: 20 partitions in the table:
===================================================================================
ID| name            |          offset|(   human)|            size|(   human)| masks
-----------------------------------------------------------------------------------
 0: bootloader                      0 (   0.00B)           400000 (   4.00M)      0
    (GAP)                                                 2000000 (  32.00M)
 1: reserved                  2400000 (  36.00M)          4000000 (  64.00M)      0
    (GAP)                                                  800000 (   8.00M)
 2: cache                     6c00000 ( 108.00M)         46000000 (   1.09G)      2
    (GAP)                                                  800000 (   8.00M)
 3: env                      4d400000 (   1.21G)           800000 (   8.00M)      0
    (GAP)                                                  800000 (   8.00M)
 4: logo                     4e400000 (   1.22G)           800000 (   8.00M)      1
    (GAP)                                                  800000 (   8.00M)
 5: recovery                 4f400000 (   1.24G)          1800000 (  24.00M)      1
    (GAP)                                                  800000 (   8.00M)
 6: misc                     51400000 (   1.27G)           800000 (   8.00M)      1
    (GAP)                                                  800000 (   8.00M)
 7: dtbo                     52400000 (   1.29G)           800000 (   8.00M)      1
    (GAP)                                                  800000 (   8.00M)
 8: cri_data                 53400000 (   1.30G)           800000 (   8.00M)      2
    (GAP)                                                  800000 (   8.00M)
 9: param                    54400000 (   1.32G)          1000000 (  16.00M)      2
    (GAP)                                                  800000 (   8.00M)
10: boot                     55c00000 (   1.34G)          1000000 (  16.00M)      1
    (GAP)                                                  800000 (   8.00M)
11: rsv                      57400000 (   1.36G)          1000000 (  16.00M)      1
    (GAP)                                                  800000 (   8.00M)
12: metadata                 58c00000 (   1.39G)          1000000 (  16.00M)      1
    (GAP)                                                  800000 (   8.00M)
13: vbmeta                   5a400000 (   1.41G)           200000 (   2.00M)      1
    (GAP)                                                  800000 (   8.00M)
14: tee                      5ae00000 (   1.42G)          2000000 (  32.00M)      1
    (GAP)                                                  800000 (   8.00M)
15: vendor                   5d600000 (   1.46G)         14000000 ( 320.00M)      1
    (GAP)                                                  800000 (   8.00M)
16: odm                      71e00000 (   1.78G)          8000000 ( 128.00M)      1
    (GAP)                                                  800000 (   8.00M)
17: system                   7a600000 (   1.91G)         74000000 (   1.81G)      1
    (GAP)                                                  800000 (   8.00M)
18: product                  eee00000 (   3.73G)          8000000 ( 128.00M)      1
    (GAP)                                                  800000 (   8.00M)
19: data                     f7600000 (   3.87G)       1c27600000 ( 112.62G)      4
===================================================================================
```
Or you can use the ``dsnapshot`` and ``esnapshot`` mode, which will give you three easy-to-parse partition info snapshots on stdout (all ampart logs are printed to stderr, stdout is reserved for snapshots):  
或者你可以用``dsnapshot``和``esnapshot``模式，来在标准输出上获得三个易于处理的的分区信息快照（ampart的所有日志都打印在标准错误上，标准输出保留给快照使用）
```
ampart /dev/your_eMMC_drive --mode dsnapshot
```
*DTB snapshot decimal, script-friendly  
DTB分区快照，十进制，脚本友好*
```
logo::8388608:1 recovery::25165824:1 misc::8388608:1 dtbo::8388608:1 cri_data::8388608:2 param::16777216:2 boot::16777216:1 rsv::16777216:1 metadata::16777216:1 vbmeta::2097152:1 tee::33554432:1 vendor::335544320:1 odm::134217728:1 system::1946157056:1 product::134217728:1 cache::1174405120:2 data::-1:4 
```
*DTB snapshot hex-decimal, script-friendly  
DTB分区快照，十六进制，脚本友好*
```
logo::0x800000:1 recovery::0x1800000:1 misc::0x800000:1 dtbo::0x800000:1 cri_data::0x800000:2 param::0x1000000:2 boot::0x1000000:1 rsv::0x1000000:1 metadata::0x1000000:1 vbmeta::0x200000:1 tee::0x2000000:1 vendor::0x14000000:1 odm::0x8000000:1 system::0x74000000:1 product::0x8000000:1 cache::0x46000000:2 data::-1:4 
```
*DTB snapshot human-readable  
DTB分区快照，人类易读*
```
logo::8M:1 recovery::24M:1 misc::8M:1 dtbo::8M:1 cri_data::8M:2 param::16M:2 boot::16M:1 rsv::16M:1 metadata::16M:1 vbmeta::2M:1 tee::32M:1 vendor::320M:1 odm::128M:1 system::1856M:1 product::128M:1 cache::1120M:2 data::-1:4
```
*EPT snapshot, decimal, script-friendly  
EPT快照，十进制，脚本友好*
```
bootloader:0:4194304:0 reserved:37748736:67108864:0 cache:113246208:1174405120:2 env:1296039936:8388608:0 logo:1312817152:8388608:1 recovery:1329594368:25165824:1 misc:1363148800:8388608:1 dtbo:1379926016:8388608:1 cri_data:1396703232:8388608:2 param:1413480448:16777216:2 boot:1438646272:16777216:1 rsv:1463812096:16777216:1 metadata:1488977920:16777216:1 vbmeta:1514143744:2097152:1 tee:1524629504:33554432:1 vendor:1566572544:335544320:1 odm:1910505472:134217728:1 system:2053111808:1946157056:1 product:4007657472:134217728:1 data:4150263808:120919687168:4 
```
*EPT snapshot, hex-decimal, script-friendly  
EPT快照，十六进制，脚本友好*
```
bootloader:0x0:0x400000:0 reserved:0x2400000:0x4000000:0 cache:0x6c00000:0x46000000:2 env:0x4d400000:0x800000:0 logo:0x4e400000:0x800000:1 recovery:0x4f400000:0x1800000:1 misc:0x51400000:0x800000:1 dtbo:0x52400000:0x800000:1 cri_data:0x53400000:0x800000:2 param:0x54400000:0x1000000:2 boot:0x55c00000:0x1000000:1 rsv:0x57400000:0x1000000:1 metadata:0x58c00000:0x1000000:1 vbmeta:0x5a400000:0x200000:1 tee:0x5ae00000:0x2000000:1 vendor:0x5d600000:0x14000000:1 odm:0x71e00000:0x8000000:1 system:0x7a600000:0x74000000:1 product:0xeee00000:0x8000000:1 data:0xf7600000:0x1c27600000:4 
```
*EPT snapshot, human-readable  
EPT快照，人类易读*
```
bootloader:0B:4M:0 reserved:36M:64M:0 cache:108M:1120M:2 env:1236M:8M:0 logo:1252M:8M:1 recovery:1268M:24M:1 misc:1300M:8M:1 dtbo:1316M:8M:1 cri_data:1332M:8M:2 param:1348M:16M:2 boot:1372M:16M:1 rsv:1396M:16M:1 metadata:1420M:16M:1 vbmeta:1444M:2M:1 tee:1454M:32M:1 vendor:1494M:320M:1 odm:1822M:128M:1 system:1958M:1856M:1 product:3822M:128M:1 data:3958M:115318M:4
```
*All these snapshots can be used in corresponding clone mode (``dclone``, ``eclone``) directly, which will guarantee to revert the partitions to what they were like when the snapshots were taken  
所有这些快照都能直接用在对应的克隆模式里（``dclone``, ``eclone``），克隆模式能保证把分区恢复到和快照的时候一模一样的样子*

To restore a DTB snapshot  
恢复一个DTB快照
```
ampart /dev/your_eMMC_drive --mode dclone logo::8M:1 recovery::24M:1 misc::8M:1 dtbo::8M:1 cri_data::8M:2 param::16M:2 boot::16M:1 rsv::16M:1 metadata::16M:1 vbmeta::2M:1 tee::32M:1 vendor::320M:1 odm::128M:1 system::1856M:1 product::128M:1 cache::1120M:2 data::-1:4
```

To restore an EPT snapshot  
恢复一个EPT快照
```
ampart /dev/your_eMMC_drive --mode eclone bootloader:0B:4M:0 reserved:36M:64M:0 cache:108M:1120M:2 env:1236M:8M:0 logo:1252M:8M:1 recovery:1268M:24M:1 misc:1300M:8M:1 dtbo:1316M:8M:1 cri_data:1332M:8M:2 param:1348M:16M:2 boot:1372M:16M:1 rsv:1396M:16M:1 metadata:1420M:16M:1 vbmeta:1444M:2M:1 tee:1454M:32M:1 vendor:1494M:320M:1 odm:1822M:128M:1 system:1958M:1856M:1 product:3822M:128M:1 data:3958M:115318M:4
```

A recommended way to utilize it, is the dclone mode which will manipulate the dtb and also update the table stored on eMMC, e.g.  
建议使用的方式，是用dclone模式，这个模式会修改DTB并且也更新eMMC里面的分区表，比如
```
ampart /dev/your_eMMC_drive --mode dclone data::-1:4
```
This will modify the partitions node in DTB, remove all other partitions and create a single data partition which will fill the table up, then update EPT according to the partitions node in DTB, and automatically migrate essential partitions (e.g. env) so you don't need to dd before and after to backup and restore them  
这条命令会修改DTB里面的分区节点，删除所有分区，只创建一个data分区来自动填充分区表，然后根据DTB里面的分区表来更新EPT，然后自动迁移所有重要的分区（比如env），你就不用在前后用dd分别备份和还原了

*result EPT on my HK1Box  
我HK1Box上的结果的EPT*
```
===================================================================================
ID| name            |          offset|(   human)|            size|(   human)| masks
-----------------------------------------------------------------------------------
 0: bootloader                      0 (   0.00B)           400000 (   4.00M)      0
    (GAP)                                                 2000000 (  32.00M)
 1: reserved                  2400000 (  36.00M)          4000000 (  64.00M)      0
    (GAP)                                                  800000 (   8.00M)
 2: cache                     6c00000 ( 108.00M)                0 (   0.00B)      0
    (GAP)                                                  800000 (   8.00M)
 3: env                       7400000 ( 116.00M)           800000 (   8.00M)      0
    (GAP)                                                  800000 (   8.00M)
 4: data                      8400000 ( 132.00M)       1d16800000 ( 116.35G)      4
===================================================================================
```
Then the ``env`` partition will be guaranteed to be placed at 116M~124M. And you can freely create partition in your MBR table at 4M~36M (gap between bootloader and reserved), 100M~116M (gap between reserved and env, since cache is 0B), 117M~end (since env starts at 116M and the actual data size in env is usually only 64K, it's safe enough to begin at 117M)  
这样的话，``env``分区就一定会被放置在116M到124M的位置。你可以在你的MBR分区表里自由地在4M-36M（bootloader和reserved之间的空隙），100M-116M（reserved和env之间的空隙，因为cache大小是0），117M到结尾（因为env在116M开始，它里面的实际数据大小一般只有64K，从117M开始是足够安全的）这些地方创建分区

Even better, if you have the Android burning image and UART connection, you can unpack that Android .img for you device, then use ``ampart`` to delete all partitions except the last ``data`` partition in the DTB (if it's multi/gzipped DTB, corresponding operations are needed), and also those partition files you don't need in the image itself:  
甚至更棒的是，如果你有安卓刷机包，也有UART连接，你可以把安卓的.img解包，然后用``ampart``删掉DTB里面除了最后的``data``分区之外的所有分区（如果DTB是多合一/gzip压缩过的，也需要对应的操作），再把你不需要的分区文件都删掉
```
ampart meson1.dtb_or_aml_dtb.PARTITION --mode dclone data::-1:4
rm -f boot.PARTITION system.PARTITION .....
vi image.cfg # Also remove these partitions from it
```
Then if you re-pack the image, you'll have a burning image that's only several MiBs in size and very fast to burn. Since you don't need anything for Android this powerful image is enough to set you sailing for the external Linux boot you'll only need to use.  
然后再把镜像包打包起来，你就有了一个只有几MiB大小，刷起来很快的刷机包了。因为你根本不需要任何安卓的东西，这个强大的包足够让你准备好你只需要的外部Linux启动的环境了

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
[part emmc]: #extra-partitioning-of-emmc--给emmc分区
[bin AUR]: #extra-prebuilt-aur-packages提前打包好的aur包
[setups]: #setups--安装方法

[ophub's Armbian]: https://github.com/ophub/amlogic-s9xxx-armbian/releases
[Arch Wiki Doc Formatting]: https://wiki.archlinux.org/title/File_systems#Create_a_file_system
[arch-install-scripts]: https://archlinuxarm.org/packages/any/arch-install-scripts
[pacman]: https://archlinuxarm.org/packages/aarch64/pacman
[rootfs main]: http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
[rootfs tuna]: https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/os/ArchLinuxARM-aarch64-latest.tar.gz
[man fstab]: https://man.archlinux.org/man/fstab.5
[AUR firmware]: https://aur.archlinux.org/packages/linux-firmware-amlogic-ophub
[AUR kernel]: https://aur.archlinux.org/packages/linux-aarch64-flippy-bin
[AUR hook]: https://aur.archlinux.org/packages/uboot-legacy-initrd-hooks
[AUR ampart]: https://aur.archlinux.org/packages/ampart-git
[boot flow]: https://u-boot.readthedocs.io/en/latest/board/amlogic/boot-flow.html
[ophub bl]: https://github.com/ophub/amlogic-s9xxx-armbian/tree/main/build-armbian/amlogic-u-boot
[ampart]: https://github.com/7Ji/ampart
[build u-boot]: https://u-boot.readthedocs.io/en/latest/board/amlogic/odroid-c4.html
[amlogic-s9xxx-armbian]: https://github.com/ophub/amlogic-s9xxx-armbian
[gxlimg]: https://github.com/repk/gxlimg
[unifreq's u-boot]: https://github.com/unifreq/u-boot
[unifreq's amlogic-boot-fip]: https://github.com/unifreq/amlogic-boot-fip
[syslinux config]: https://wiki.syslinux.org/wiki/index.php?title=Config
[ampart-git]: https://aur.archlinux.org/packages/ampart-git
[AUR bin mirror]: https://ee.fuckblizzard.com/AUR