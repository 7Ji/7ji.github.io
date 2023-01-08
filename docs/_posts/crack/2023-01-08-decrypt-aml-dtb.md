---
layout: post
title:  "Extracting encrypted DTBs from Amlogic boxes so ampart can work on them / 提取Amlogic盒子上的加密DTB从而让ampart可以在其上工作"
date:   2023-01-08 10:00:00 +0800
categories: crack
---

## Applicability / 适用性

This is only for extraction of DTBs on devices with **encrypted** DTBs, [ampart] can work on boxes with non-encrypted DTBs directly. You would only need to do this if running [ampart] result in a failure with the following log:  
此提取方法只有**DTB加密**的盒子才需要，对于不加密的盒子，ampart可以直接工作。仅当你运行ampart出现下面的报错时才需要此方法：
```
DTB read partitions and report: Unrecognizable DTB, magic 9d8c8056
```

This is only for experienced Amlogic device tinkers. If done incorrectly you could brick your device and have to debrick it with a stock Android firmware, which will result in loss of your data on eMMC, **you have been warned and take your own risk**.  
本方法只适用于经验老到的Amlogic设备折腾玩家。如果操作不当，盒子可能会变砖，且你只能通过原厂安卓固件来解砖，并丧失eMMC上的所有数据，**后果自负**

## Requirements / 需求
 - Bootable Android firmware on the box with encrypted DTBs  
 盒子上有可启动的安卓固件，DTB加密
 - Bootable external Linux system on the box where [ampart] is executable  
 盒子上有可启动的外部Linux系统，可以运行[ampart]
 - ADB access through network on Android  
 安卓下有ADB网络访问
 - A Linux machine seperate from the box for operation where `adb` and `dtc` are available  
 独立于这个盒子的用来操作的linux机器，可以使用`adb`和`dtc`



## Steps / 步骤
 1. Boot the box into Android and make sure ADB is enabled  
 启动盒子到安卓，并确定ADB已启用
 2. Connect to the box through ADB  
通过ADB连接到盒子
    ```
    adb connect [your box IP, e.g. 192.168.7.47]
    ```
 3. Pull the decrypted device-tree folder under psuedo FS `/proc` to the current work folder  
 把伪文件系统`/proc`下的解密的设备树文件夹拉取到当前工作目录
    ```
    adb pull /proc/device-tree .
    ```
    *The device-tree must be decypted for the Android/Linux kernel to function, thus a running Android system must be able to **see** this decrypted folder, so we could extract it like this  
    要让Android/Linux内核正常工作，设备树必须被解密，因此一个正在运行的安卓系统一定能**看到**这个解密过的文件夹，这就是我们可以这样提取的原因*
 4. Wait till the command to finish, it could take ~1Min, when finished the log should read like this:  
 等待命令完成，大约需要一分钟，完成以后会有这样的日志：
    ```
    /proc/device-tree/: 2917 files pulled, 0 skipped. 0.0 MB/s (30864 bytes in 39.501s)
    ```
 5. Convert the plain device-tree folder to decrypted DTB  
 把明文的设备树文件夹转化为解密的DTB
    ```
    dtc -I fs device-tree -o decrypted.dtb
    ```
 6. If the DTB is larger than 262128 (0x3fff0, 256KiB-16B) Bytes, gzipped it up, this is usually not needed as most DTBs are under 100KiB  
 如果DTB大于262128(0x3fff0, 256KiB-16B)字节，用gzip压缩，通常并不需要，因为绝大多数DTB小于100KiB
 7. Boot the box into the external Linux system where [ampart] is executable  
 启动盒子到可以运行[ampart]的外部Linux系统
 8. Transfer the decrypted DTB to the box via SSH, SCP Samba or physical medias  
 把解密的DTB通过SSH，SCP, Samba或者物理媒介传输到盒子上
 9. Identify the eMMC block device, which should be `/dev/mmcblk0` if you're running systems with Amlogic kernel, e.g. EmuELEC, etc; or `/dev/mmcblk2` if you're running systems with mainline kernel, e.g. ArchLinuxARM, Armbian, OpenWrt, etc.  
 辨别eMMC的块设备，如果你运行的系统使用的是Amlogic内核，比如EmuELEC等，那么应该是`/dev/mmcblk0`；如果你运行的系统使用的是主线内核，比如ArchLinuxARM，Armbian，OpenWrt等，那么应该是`/dev/mmcblk2`
 10. Write the DTB to eMMC at 40MiB offset and 40MiB+256KiB offset seperately. *The DTBs are stored at 40MiB ~ 40MiB+256KiB, and 40MiB+256KiB ~ 40MiB+512KiB, as two identical copies, the last 16Bytes of both are 4-Byte magic, version, timestamp and checksum*  
 把DTB分别写入到eMMC的偏移40MiB和40MiB+256KiB。*DTB存储在40MiB到40MiB+256KiB，和40MiB+256KiB到40MiB+512KiB，两份完全相同，两者最后的16字节都依次是4字节的幻数，版本，时间戳和校验和*
      ```
      dd if=decrypted.dtb of=[your eMMC block device] bs=256K seek=160 conv=notrunc
      dd if=decrypted.dtb of=[your eMMC block device] bs=256K seek=161 conv=notrunc
      sync
      ```
     *If you're writing to the reserved block device (`/dev/reserved`) instead which is only available under Amlogic kernel, then the seek numbers should be 16 and 17  
     如果你是在只有Amlogic内核下才有的保留块设备上操作（`/dev/reserved`），后寻数应该是16和17*  
     *We write to both copies and do not update the checksums, because if both copies are considered 'corrupted' (checksum mismatch), Amlogic's u-boot will still use the first one. The second copy is written to purposedly make it also 'corrupted' so the u-boot won't use the stock encrypted one.  
     我们往两份里都写入了，并且没有更新校验和，因为如果两份都被认为'损坏'（校验和不符）了的话，Amlogic的u-boot仍然会用第一份。第二份写入的目的是刻意让它也'损坏'，从而让u-boot不会用原厂的加密DTB*
 11. Reboot to check if the box still boots. And if it does, you can use [ampart] to partition it however you like it.  
 重启，检查盒子是否仍能启动。如果能启动的话，你就能用[ampart]来随心所欲地分区了。

## Special thanks / 特别鸣谢
Special thanks to Tomasz Jaskólski in helping with testing with the method documented here. I don't have a box with encrypted DTB so it's impossible for me to test it by myself and post it.   
特别鸣谢Tomasz Jaskólski帮助测试上述方法。我自己没有有加密DTB的盒子，所以没法亲自测试并发布这个方法。

[ampart]: https://github.com/7Ji/ampart
