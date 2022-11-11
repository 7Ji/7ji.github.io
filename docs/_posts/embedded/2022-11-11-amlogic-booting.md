---
layout: post
title:  "Bootflow and configuration on Amlogic device / Amlogic设备上的启动流程和配置"
date:   2022-11-11 10:18:07 +0800
categories: embedded
---

*This article is splitted from [Installation of ArchLinux ARM on an out-of-tree Amlogic device][alarm-install] for a better structure. Check that article if you are interesting on an ArchLinux ARM installation done in the ArchLinux way.  
这篇文章是从[在不被官方支持的Amlogic设备上安装ArchLinux ARM][alarm-install]中为了更好的布局而分割出来的。如果你对以ArchLinux的方式安装ArchLinux ARM感兴趣，看一下那篇文章*

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
|efi-stub*|-|yes|depends|no|standard
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
This script expects a plain-text file ``uEnv.txt`` with ``key=value`` pairs to provide essential variables:  
这个脚本期待一个里面是``键=值``对的纯文本文件``uEnv.txt``来提供必要的变量
```
LINUX=/zImage
INITRD=/uInitrd
FDT=/dtb/amlogic/meson-sm1-x96-max-plus-100m.dtb
APPEND=root=UUID=5eaeec73-cfda-47de-acd3-e6dc43f7c13e rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1
```
Note ``INITRD`` here points to a legacy uboot initrd, this is due to the command ``booti`` is used in the script to load the kernel, for which the initramfs must be in the format of legacy uboot initrd  
注意这里的``INITRD``指向到了一个传统的uboot初始化内存盘，这是因为脚本里是用命令``booti``来加载内核的，这条命令要求初始化内存盘必须以uboot传统初始化内存盘的形式存储

As you can see the file has a syntax similar to shell scripting, since u-boot uses hush shell and some common shell syntaxes are ported, but you won't find your familiar shell tools and couldn't use fancy stuffs like piping.  
如同所见，这个文件的语法和shell脚本很相似，因为u-boot是用的是hush shell，然后一些常见的shell语法被移植了，不过你找不到熟悉的shell工具，并且也不能整一些诸如管道之类的花活

Different from syslinux config, you have do all kinds of different things in a single script, as long as it's supported by u-boot itself. So a boot target in u-boot can then boot multiple targets in script thanks to some basic loops.  
和syslinux配置不同，你可以在一个脚本里做各种各样的事情，只要u-boot本身能支持。所以u-boot里的单一启动目标可以启动多个启动目标，多亏了一些基本的循环逻辑

The plain-text boot script can then be compiled into a u-boot script with the following command  
纯文本的启动脚本可以通过下面的命令来编译成u-boot脚本
```
mkimage -C none -A arm -T script -d /path/to/plain/text/script /path/to/uboot/script
```

### efi-stub
This is a special case for mainline u-boot, sinec u-boot added EFI support long time ago, it's possible to use ``bootefi`` instead of ``booti`` and ``sysboot`` to load an EFI-STUB, which could be the kernel itself or another bootloader like GRUB, systemd-boot. If, e.g. kernel, initrd and fdts are all in eMMC partition 4, then the following commands can load the kernel directly if it was built with ``CONFIG_EFI_STUB=y``:  
这是一个主线u-boot的特别情况，因为u-boot很久以前就添加了EFI支持，可以使用``bootefi``而不是``booti``和``sysboot``来加载一个EFI执行程序，这个EFI执行程序既可以是内核自己，也可以是像GRUB，systemd-boot这样的引导程序。比如，当内核，初始化内存盘，和设备树都放在eMMC分区4的时候，下面的命令就可以直接加载一个构建时配置了``CONFIG_EFI_STUB=y``的内核：

```
load mmc 1:4 ${fdt_addr_r} /boot/dtbs/linux-aarch64-flippy/amlogic/meson-gxl-s905x-p212.dtb
load mmc 1:4 ${kernel_addr_r} /boot/vmlinuz-linux-aarch64-flippy
setenv bootargs 'initrd=/boot/initramfs-linux-aarch64-flippy.img root=UUID=9f75c223-4ebe-48b7-b4cb-f3823ed5f96f rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes'
bootefi ${kernel_addr_r} ${fdt_addr_r}
```
**Note the argument initrd should be passed as an argument to kernel directly, and the path should be seperated by ``/`` instead of ``\`` like on x86-64. The initrd *has to be a standard initramfs and not a u-boot legacy initrd*  
注意参数initrd应该被作为参数穿递给内核，并且路径应当以``/``而不是像x86-64上那样以``\``分割。初始化内存盘*必需是标准的initramfs，而不可以是u-boot传统内存盘***

This can be encapsuled with a booting script and an external plain text file to easily update the essential variables  
可以通过一个启动脚本和一个外部的纯文本文件来方便地更新必要变量，来把这一启动逻辑封装

e.g. The bare-minumum source for ``/boot/boot.scr`` (remember to compile it to u-boot script) when these are loaded from eMMC part 4  
比如，最基本的``/boot/boot.scr``源码（记得把它编译为u-boot脚本），当这些东西都是从eMMC分区4加载的时候
```
load mmc 1:4 ${loadaddr} /boot/uEnv.txt
env import -t ${loadaddr}
load mmc 1:4 ${fdt_addr_r} ${FDT}
load mmc 1:4 ${kernel_addr_r} ${LINUX}
setenv bootargs "initrd=${INITRD} ${APPEND}"
bootefi ${kernel_addr_r} ${fdt_addr_r}
```
with its corresponding plain-text``uEnv.txt``:  
它所对应的纯文本的``uEnv.txt``
```
LINUX=/boot/vmlinuz-linux-aarch64-flippy
INITRD=/boot/initramfs-linux-aarch64-flippy.img
FDT=/boot/dtbs/linux-aarch64-flippy/amlogic/meson-gxl-s905x-p212.dtb
APPEND=root=UUID=9f75c223-4ebe-48b7-b4cb-f3823ed5f96f rootflags=data=writeback rw rootfstype=ext4 console=ttyAML0,115200n8 console=tty0 fsck.fix=yes fsck.repair=yes
```
*Under the hood this script will be executed by a built-in u-boot variable like this:  
实际上这个脚本会被u-boot里的内置变量以下面这样的形式来执行*
```
load mmc 1:4 ${scriptaddr} /boot/boot.scr
source ${scriptaddr}
```

When booting take note of the u-boot log before kernel is loaded to verify it's booted as EFI stub:  
启动的时候注意看u-boot的日志，来确认它以EFI可执行程序被启动了
```
Booting /\boot\vmlinuz-linux-aarch64-flippy
EFI stub: Booting Linux Kernel...
EFI stub: Using DTB from configuration table
EFI stub: Loaded initrd from command line option
EFI stub: Exiting boot services...
```
And check sysfs in userspace to confirm EFIvars are identified  
并且在用户空间里检查sysfs来确认EFI变量被识别了
```
# ls /sys/firmware/efi/efivars/
AuditMode-8be4df61-93ca-11d2-aa0d-00e098032b8c               PlatformLang-8be4df61-93ca-11d2-aa0d-00e098032b8c       SetupMode-8be4df61-93ca-11d2-aa0d-00e098032b8c
DeployedMode-8be4df61-93ca-11d2-aa0d-00e098032b8c            PlatformLangCodes-8be4df61-93ca-11d2-aa0d-00e098032b8c  VendorKeys-8be4df61-93ca-11d2-aa0d-00e098032b8c
OsIndicationsSupported-8be4df61-93ca-11d2-aa0d-00e098032b8c  SecureBoot-8be4df61-93ca-11d2-aa0d-00e098032b8c
```
You could in theory load other boot managers as EFISTUB but as I think that just adds non-neccessary boot times, so you could try and configure that by yourself and I'll not write about that  
理论上你也能把其他启动管理器作为EFISTUB加载，不过我认为那样的话只是增加了不必要的启动时间，所以你可以自己去尝试和配置，我就不写了


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
or even just use ``aml_autoscript`` itself to load the kernel each time, if you blow up the on-eMMC Android system and env partition, the stock u-boot will 100% fallback to update mode, so this script will always be executed  
或者干脆用``aml_autoscript``本身来每次加载内核，如果你把eMMC上的安卓系统和env分区搞爆炸了，那么原厂的u-boot一定会回落到升级模式，那么这个脚本就总是会被执行
```
fatload mmc 1:1 0x1000000 uEnv.txt
env import -t 0x1000000
fatload mmc 1:1 0x1000000 ${FDT}
fatload mmc 1:1 0x1080000 ${LINUX}
fatload mmc 1:1 0x3080000 ${INITRD}
setenv bootargs ${APPEND}
booti 0x1080000 0x3080000 0x1000000
```

[setups]: #setups--安装方法

[alarm-install]: ../08/alarm-install.html

[amlogic-s9xxx-armbian]: https://github.com/ophub/amlogic-s9xxx-armbian
[ampart]: https://github.com/7Ji/ampart
[boot flow]: https://u-boot.readthedocs.io/en/latest/board/amlogic/boot-flow.html
[build u-boot]: https://u-boot.readthedocs.io/en/latest/board/amlogic/odroid-c4.html
[gxlimg]: https://github.com/repk/gxlimg
[ophub bl]: https://github.com/ophub/amlogic-s9xxx-armbian/tree/main/build-armbian/amlogic-u-boot
[syslinux config]: https://wiki.syslinux.org/wiki/index.php?title=Config
[unifreq's u-boot]: https://github.com/unifreq/u-boot
[unifreq's amlogic-boot-fip]: https://github.com/unifreq/amlogic-boot-fip