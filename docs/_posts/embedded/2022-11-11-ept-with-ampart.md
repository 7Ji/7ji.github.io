---
layout: post
title:  "Partitioning on Amlogic's proprietary eMMC partition table with ampart / 使用ampart在Amlogic专有的eMMC分区表上分区"
date:   2022-11-11 10:30:07 +0800
categories: embedded
---

*This article is splitted from [Installation of ArchLinux ARM on an out-of-tree Amlogic device][alarm-install] for a better structure. Check that article if you are interesting on an ArchLinux ARM installation done in the ArchLinux way.  
这篇文章是从[在不被官方支持的Amlogic设备上安装ArchLinux ARM][alarm-install]中为了更好的布局而分割出来的。如果你对以ArchLinux的方式安装ArchLinux ARM感兴趣，看一下那篇文章*

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
简单地把eMMC驱动器作为参数来调用，你就能看到EPT和DTB里面的partitions节点：
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
This will modify the partitions node in DTB, remove all other partitions and create a single data partition which will fill the table up, then update EPT according to the partitions node in DTB, and automatically migrate essential partitions (e.g. env) so you don't need to dd before and after to backup and restore them.  
这条命令会修改DTB里面的分区节点，删除所有分区，只创建一个data分区来自动填充分区表，然后根据DTB里面的分区表来更新EPT，然后自动迁移所有重要的分区（比如env），你就不用在前后用dd分别备份和还原了。


*The snapshot (``data::-1:4``) here is gotten from dedit mode, where unwanted partitions are deleted one-by-one (``^`` + name selects a partition by its name, ``?`` deletes the partition):  
这里的快照(``data::-1:4``)是通过在dedit模式下把不想要的分区一个一个删掉来得到的（`^`+名字来按名字选择分区，`?`删掉分区）*
```
ampart /dev/mmcblk2 --mode dedit ^recovery? ^dtbo? ^cri_data? ^param? ^boot? ^rsv? ^metadata? ^vbmeta? ^tee? ^vendor? ^odm? ^system? ^product? ^cache? --dry-run
```

*The above dedit session would convert the EPT to like this, where ``logo`` and ``misc`` is kept, so during booting you could still see the logo, and some misc things set by Android in ``misc`` partition could still be effective:  
上边的命令会把EPT转化成这样，`logo`和`misc`得以保存，这样在启动的时候你仍然能看到logo，安卓在`misc`分区里设置的一些杂项也仍然会有效*
```
DTS report partitions: 3 partitions in the DTB:
=======================================================
ID| name            |            size|(   human)| masks
-------------------------------------------------------
 0: logo                       800000 (   8.00M)      1
 1: misc                       800000 (   8.00M)      1
 2: data                              (AUTOFILL)      4
=======================================================
```
```
EPT report: 7 partitions in the table:
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
 4: logo                      8400000 ( 132.00M)           800000 (   8.00M)      1
    (GAP)                                                  800000 (   8.00M)
 5: misc                      9400000 ( 148.00M)           800000 (   8.00M)      1
    (GAP)                                                  800000 (   8.00M)
 6: data                      a400000 ( 164.00M)       1d14800000 ( 116.32G)      4
===================================================================================
```
*But if you just want say goodbye to all the stuffs not related to the Linux distro you're going to use, you can go further to delete even the ``logo`` and ``misc`` partitions:  
但如果你想对所有和你之后要用的Linux发行版无关的东西说拜拜，你可以再进一步，甚至把``logo``和``misc``分区删了*
```
ampart /dev/mmcblk2 --mode dedit ^logo? ^misc?
```

*Then this will get you a DTB with only data partition, and an EPT with only several partitions:  
这样的话你就能得到只有data分区的DTB，并且只有几个分区的EPT：*
```
DTS report partitions: 1 partitions in the DTB:
=======================================================
ID| name            |            size|(   human)| masks
-------------------------------------------------------
 0: data                              (AUTOFILL)      4
=======================================================
```
```
EPT report: 5 partitions in the table:
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
*And if you take a snapshot in ``dsnapshot`` mode, you'll get the snapshot (`data::-1:4`) mentioned above  
如果你在``dsnapshot``模式下进行快照的话，你就能拿到上面提到的快照（`data::-1:4`）*  

*For post-SC2(S905X4, S905D4) devices like HK1 RBOX X4, there're more partitions you shouldn't remove, the snapshot should be changed as to:  
对于SC2(S905X4, S905D4)之后的设备，比如说HK1 RBOX X4，有更多不能移除的分区，快照应该改成这个样子*
```
boot_a::64M:1 vbmeta_a::2M:1 vbmeta_system_a::2M:1 data::-1:4 
```

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

This can be pushed further to extreme with another run in ``ecreate`` mode, which will remove the partitions node in DTB so the EPT won't be "fixed" by Amlogic u-boot, so ampart can have more freedom on the placement of partitions.   
通过再用一次``ecreate``模式，把DTB里面的分区节点直接移除，废掉Amlogic u-boot的“修复”EPT功能，ampart就能在分区的布局上有更大的自由。

**This mode should not be used in post-SC2(s905x4,s905d4) devices and several specific early devices with specific firmware (e.g. R3300L with milton's Android 6.0 firmware), some of them on partitions node in DTB to get env partitions to get persistent booting configuration but can still boot with `aml_autoscript` when it's broken (e.g. R3300L), others won't even boot (e.g. HK1 RBOX X4)  
不要在SC2（s905x4，s905d4）之后的设备和一些有特定固件版本的特定设备（比如用milton的安卓6.0固件的R3300L）上用这个模式，它们中有的依赖于DTB中的分区节点来获得环境分区从而读取持久化的启动数据，不过即使坏了也能通过`aml_autoscript`来启动（比如R3300L），有的直接就不能启动（比如HK1 RBOX X4）**
```
ampart /dev/your_eMMC_drive --mode ecreate data:::
```
And the EPT will become like this, more spaces can be used in MBR partition table(4-36M, 100M-end):  
EPT分区表就会变成这样，更多的空间可以在MBR分区表里分配（5-36M, 100M到结尾）
```
DTB get partitions: partitions node does not exist in dtb
```
```
EPT report: 5 partitions in the table:
===================================================================================
ID| name            |          offset|(   human)|            size|(   human)| masks
-----------------------------------------------------------------------------------
 0: bootloader                      0 (   0.00B)           400000 (   4.00M)      0
 1: env                        400000 (   4.00M)           800000 (   8.00M)      0
 2: cache                      c00000 (  12.00M)                0 (   0.00B)      0
    (GAP)                                                 1800000 (  24.00M)
 3: reserved                  2400000 (  36.00M)          4000000 (  64.00M)      0
 4: data                      6400000 ( 100.00M)       1d18800000 ( 116.38G)      4
===================================================================================
```

Even better, if you have the Android burning image and UART connection, you can unpack that Android .img for you device, then use ``ampart`` to delete all partitions except the last ``data`` partition in the DTB (if it's multi/gzipped DTB, corresponding operations are needed), and also those partition files you don't need in the image itself:  
甚至更棒的是，如果你有安卓刷机包，也有UART连接，你可以把安卓的.img解包，然后用``ampart``删掉DTB里面除了最后的``data``分区之外的所有分区（如果DTB是多合一/gzip压缩过的，也需要对应的操作），再把你不需要的分区文件都删掉
```
ampart meson1.dtb_or_aml_dtb.PARTITION --mode dclone data::-1:4
rm -f boot.PARTITION system.PARTITION .....
vi image.cfg # Also remove these partitions from it
```
Then if you re-pack the image, you'll have a burning image that's only several MiBs in size and very fast to burn. Since you don't need anything for Android this powerful image is enough to set you sailing for the external Linux boot you'll only need to use.  
然后再把镜像包打包起来，你就有了一个只有几MiB大小，刷起来很快的刷机包了。因为你根本不需要任何安卓的东西，这个强大的包足够让你准备好你只需要的外部Linux启动的环境了


[alarm-install]: ../08/alarm-install.html

[ampart]: https://github.com/7Ji/ampart
[ampart-git]: https://aur.archlinux.org/packages/ampart-git