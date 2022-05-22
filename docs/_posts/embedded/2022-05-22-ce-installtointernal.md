---
layout: post
title:  "How to install CoreELEC/EmuELEC to emmc if cemmc does not work"
date:   2022-05-22 14:50:07 +0800
categories: crack
---
First I would appreciate Team CoreELEC for the ceemmc tool that can easily install CoreELEC to emmc, but that tool does not work for me (ceemmc -x and segmentation fault) so I had to come up with my own method to install

If ``ceemmc`` works for you, then you should stick with it as it's the official method approved by Team CoreELEC

This tutorial only works on Amlogic-ng releases of CoreELEC (any release after v19, or those with -ng in there name before v19), if you are using old Amlogic no ng devices, you should stick with ``installtointernal``

Following this tutorial will render your Android installation **totally unusable** as we completely erase Android from emmc, if this is not what your want, stop reading.

This tutorial relies on **ampart**, a partition tool I wrote in the past few days that can support Amlogic's proprietary partition format, I've opened PR to include it in EmuELEC (https://github.com/EmuELEC/EmuELEC/pull/928) but I'm not sure if CoreELEC needs it as it already has ceemmc.

This tutorial assumes you have already enabled SSH and knows a little bit about how Linux works, and following this tutorial does not guarantee you can get a bootable CoreELEC on your emmc and won't brick your box. I take no responsibility for any data loss.

1. Login to your CoreELEC box via SSH, or serial connection.
2. Make sure mmcblk0, bootloader, reserved and env are present under /dev. If not, this tutorial does not work for you.
3. Download ampart from the release page: https://github.com/7Ji/ampart/releases, at the time I'm writing this the newest release is v0.1. You would need to download **ampart.aarch64.xz** and **ampart.aarch64.xz.sha256**.
4. Check the sha256sum of **ampart.aarch64.xz** with ``sha256sum ampart.aarch64.xz``, the result should be the same with the content of **ampart.aarch64.xz.sha256**. If not, redownload it and check again.
5. Decompress ampart with ``xz -cdk ampart.aarch64.xz > ampart``
6. Add execution permission for ampart with ``chmod +x ampart``
7. Check the current partition table of the emmc with ``./ampart /dev/mmcblk0``, the output would be like:
    ````
    Path '/dev/mmcblk0' seems a device file
    Path '/dev/mmcblk0' detected as whole emmc disk
    Path '/dev/mmcblk0' is a device, getting its size via ioctl
    Disk size is 7818182656 (7.28G)
    Reading old partition table...
    Notice: Seeking 37748736 (36.00M) (offset of reserved partition) into disk
    Validating partition table...
    Partitions count: 5, GOOD √
    Magic: MPT, GOOD √
    Version: 01.00.00, GOOD √
    Checksum: calculated 3d5695ff, recorded 3d5695ff, GOOD √
    Partition table read from '/dev/mmcblk0':
    ==============================================================
    NAME                          OFFSET                SIZE  MARK
    ==============================================================
    bootloader               0(   0.00B)    400000(   4.00M)     0
      (GAP)                                2000000(  32.00M)
    reserved           2400000(  36.00M)   4000000(  64.00M)     0
      (GAP)                                 800000(   8.00M)
    cache              6c00000( 108.00M)  20000000( 512.00M)     2
      (GAP)                                 800000(   8.00M)
    env               27400000( 628.00M)    800000(   8.00M)     0
      (GAP)                                 800000(   8.00M)
    logo              28400000( 644.00M)   2000000(  32.00M)     1
      (GAP)                                 800000(   8.00M)
    recovery          2ac00000( 684.00M)   2000000(  32.00M)     1
      (GAP)                                 800000(   8.00M)
    rsv               2d400000( 724.00M)    800000(   8.00M)     1
      (GAP)                                 800000(   8.00M)
    tee               2e400000( 740.00M)    800000(   8.00M)     1
      (GAP)                                 800000(   8.00M)
    crypt             2f400000( 756.00M)   2000000(  32.00M)     1
      (GAP)                                 800000(   8.00M)
    misc              31c00000( 796.00M)   2000000(  32.00M)     1
      (GAP)                                 800000(   8.00M)
    boot              34400000( 836.00M)   2000000(  32.00M)     1
      (GAP)                                 800000(   8.00M)
    system            36c00000( 876.00M)  80000000(   2.00G)     1
      (GAP)                                 800000(   8.00M)
    data              b7400000(   2.86G) 11ac00000(   4.42G)     4
    ==============================================================
    Disk size totalling 7818182656 (7.28G) according to partition table
    Using 7818182656 (7.28G) as the disk size
    ````
    Make sure the partition table can be recognised here (after the first == line and before the == last line), as long as your partition table can be recognized, it should be fine.  

    If ampart complains about wrong magic, you may try your luck with operating on ``/dev/reserved`` instead, as certain OEMs like Xiaomi have modified the offset, you would need to replace ``/dev/mmcblk0`` with ``/dev/reserved`` in most of the following commands, **OR** add ``--offset [offset]``, in which [offset] is the offset of reserved partition you get from the partition table, e.g. ``--offset 4M`` for Xiaomi devices (default is ``--offset 36M`` and you don't need to modify it if the part table can be recognized without it). **Don't do both** (i.e. add --offset for /dev/reserved, as it does not work)
8. Make sure none of the partitions on emmc are mounted, check results of ``grep '^/dev/' /proc/mounts``, you don't want any partations you saw in the partition table present in this command's output
9. Backup the env partition as it will be most likely moved by ampart, since there's most likely a cache partition before env partition that we do not want: ``dd if=/dev/env of=env.img`` 
10. Create a new partition table on the emmc ``./ampart /dev/mmcblk0 system::512M:2 data:::``, ampart should report a new partition table like the next one:
    ````
    ==============================================================
    NAME                          OFFSET                SIZE  MARK
    ==============================================================
    bootloader               0(   0.00B)    400000(   4.00M)     0
      (GAP)                                2000000(  32.00M)
    reserved           2400000(  36.00M)   4000000(  64.00M)     0
    env                6400000( 100.00M)    800000(   8.00M)     0
    system             6c00000( 108.00M)  20000000( 512.00M)     2
    data              26c00000(   2.11G) 1ab400000(   6.68G)     4
    ==============================================================
    ````
    (You can reduce the size of system partition if you want to squeeze every last byte from your emmc, as long as it can store all the data you currently have under /flash)
11. Restore the env partition ``dd if=env.img of=/dev/env``
12. Create a fat fs on /dev/system: ``mkfs.fat /dev/system``
13. Create a temporary mount point for system partition: ``mkdir -p /tmp/system``
14. Mount system as RW: ``mount -o rw /dev/system /tmp/system``
15. Copy all the stuff in COREELEC partition (/flash) to the new system partition ``cp -rva /flash/* /tmp/system``
16. Edit cfgload under the new system partition, using ``vi`` or ``nano`` to edit ``/tmp/system/cfgload``, find line 11 (*if test "${ce_on_emmc}" = "yes"; then setenv rootopt "BOOT_IMAGE=kernel.img boot=LABEL=CE_FLASH disk=FOLDER=/dev/CE_STORAGE"; fi*), you want to change ``boot=LABEL=CE_FLASH`` to ``boot=/dev/system`` and ``disk=FOLDER=/dev/CE_STORAGE`` to ``disk=/dev/data``.  
**NOTE** You can change some steps if you want to use the official bootargs, you need to create a partition named CE_STORAGE instead of data, put things under coreelc_storage subfolder instead of the root of CE_STORAGE, and give fat fs on /dev/system a label CE_FLASH. But since we are running pure CoreELEC I don't like the FOLDER=/dev/CE_STORAGE style
17. Sync disk I/O and umount system partition: ``sync; umount /tmp/system``
18. Create an ext4 fs on /dev/data: ``mke2fs -F -q -t ext4 -m 0 /dev/data``
19. Optionally copy your all data to the internal data partition   
    1. Stop kodi ``systemctl stop kodi``
    2. Create a temporary mount point for data partition: ``mkdir -p /tmp/data``
    3. Using rsync to transfer your data: ``rsync -qaHSx /storage/. /tmp/data``
    4. Sync disk I/O and umount data partition: ``sync; umount /tmp/data``
20. Change the stock storeboot arg since now storeboot should boot CoreELEC instead: ``/usr/sbin/fw_setenv storeboot 'run cfgloademmc'``
21. Optionally change the bootfromnand arg if you want to boot to the CoreELEC installation on emmc for your next boot: ``/usr/sbin/fw_setenv bootfromnand 1``
22. Optionally Reboot: ``reboot``

You now have a CoreELEC installtion on your emmc, enjoy it!s