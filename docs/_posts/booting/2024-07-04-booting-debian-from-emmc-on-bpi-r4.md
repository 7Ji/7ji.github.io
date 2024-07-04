---
layout: post
title:  "Booting Debian from eMMC on BPi-R4"
date:   2024-07-04 14:00:00 +0800
categories: booting
---

For quick steps without background you can jump to [Installation](#installation) section.

## Hardware

Let's focus on the hardware limitation and why flashing plain BPI official Debian image onto eMMC won't get you a bootable Debian.


### Bootflow

The MT7988 SoC on BPI-R4 as a "next-gen" "Wi-Fi 7 ready" SoC by MediaTek, its on-SoC non-flashable BootROM is hardcoded to load a BL2 from either of the following device:
- SPI storage, only SPI-NAND
- SDIO storage, either SD card or eMMC

There's no "order": the SBC came with a set of switches to change the boot device on cold boot (hot boot always boots to the same target). The following chart describes the boot targets with different switches combination:

|switch A (boot bus)|switch B (SDIO selection)|boot target|
|-|-|-|
|0|0|system halt|
|0|1|SPI NAND|
|1|0|eMMC|
|1|1|SD card|

_0 for up, 1 for down_

### BL2 and BL31 layout

For both SPI NAND and SD card boot, as there's only one hardware partition, BL2 is loaded from the exactly same storage device, and BL31 also the same device.

For eMMC boot however, BL2 is loaded from the first hardware boot partition (`mmcblk0boot0`), and BL3 then from the hardware user partition (`mmcblk0`, and it's on a different offset from SD boot). As a result, you can not write the SD card image to eMMC and expect it to work, you'll need a specific BL2 on eMMC hw boot partition.

Therefore for complete eMMC installation we could need seperate BL2 image for boot partition and eMMC image for user partition, unfortunately bananapi did not release the BL2 image for BPI-R4 for their Debian image, and that's why the later Installation steps are more complicated than expected.

### SDIO: the conflict between SD card and eMMC

The SoC only has a single SDIO port, which means there could be only one device under SDIO, and SD card is mutually exclusive to eMMC: you can connect both but there would only be one recognized. This is vastly different to Amlogic or Rockchip SoCs that I'm familiar with: on those boards you can have up to 3 different SDIO devices all working at the same time, one eMMC, one SD card, and one SDIO Wi-FI; on MT7988 you can only have one of them.

You can imagine (not really how it works) that switch A controls whether to boot from SDIO, and switch B controls which which SDIO device to boot. In reality it's only a boot selection, and SD card only works with SD card boot, not in (0,1) SPI boot. The main idea is to make it possible to install to eMMC in my opinion

## Preparation
We'll need the following images:
- A USB-TTL converter, supporting baud rate 115200
- Latest mainline OpenWRT SD card image for BPI-R4, downloaded from [OpenWRT website](https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/openwrt-mediatek-filogic-bananapi_bpi-r4-sdcard.img.gz)
- Bananapi's Debian SD card image for BPI-R4, download from [their Wiki](https://wiki.banana-pi.org/Banana_Pi_BPI-R4#Debian_11)
- An SD card for initial bootup
- A USB drive for "live USB"

## Installation

Assuming you have nothing on either SPI NAND, SD card, or eMMC, we need to go all these steps

1. Make sure BPI-R4 is powered off.
2. Connect serial console, baudrate shall be `115200n8`, keep it open and be ready to input commands or interrupt the bootflow.
3. Write latest mainline OpenWRT SD card image to SD card on your PC with your familiar tool, e.g. on Linux
    ```
    curl -L https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/openwrt-mediatek-filogic-bananapi_bpi-r4-sdcard.img.gz -o - | gzip -cd | sudo dd bs=16M of=/dev/disk/by-id/usb-YOU-CARD-READER
    ```
4. Eject the SD card from your PC safely and insert it to BPI-R4
5. Toggle BPI-R4's boot switch to 1,1 (all down) for SD boot
6. Power on, hold down arrow and wait for the U-Boot menu to show up and the auto-selection is interrupted. Make sure the menu starts with `( ( ( OpenWrt ) ) )  [SD card]`
7. Select `7. Install bootloader, recovery and production to NAND.` and press ENTER to confirm
8. Wait for the installation to finish and the following logs are shown:
    ```
    MMC read: dev # 0, block # 104448, count 16384 ... 16384 blocks read: OK
    Creating dynamic volume emmc_install of size 8388608
    8388608 bytes written to volume emmc_install
    Press ENTER to return to menu
    ```
9. Unplug the power cable to poweroff the board
10. Toggle BPI-R4's boot switch to 0,1 (left up, right down) for SPI-NAND boot
11. Power on, hold down arrow and wait for the U-Boot menu to show up and the auto-selection is interrupted. Make sure the menu starts with ` ( ( ( OpenWrt ) ) )  [SPI-NAND]`
12. Select `9. Install bootloader, recovery and production to eMMC.` and and press ENTER to confirm
13. Wait for the installation to finish and the following logs are shown:
    ```
    MMC write: dev # 0, block # 131072, count 26080 ... 26080 blocks written: OK
    Saving Environment to UBI... UBI partition 'ubi' already selected
    Writing to redundant UBI... done
    OK
    Saving Environment to UBI... UBI partition 'ubi' already selected
    Writing to UBI... done
    OK
    Press ENTER to return to menu
    ```
14. Press ENTER to return to u-boot menu, leave BPI-R4 as-is
15. Write bananapi's Debian SD card image for BPI-R4 to a USB drive on your PC with your familiar tool, e.g. on Linux
    ```
    bsdtar -xOf ~/Downloads/2024-03-10-debian-11-bullseye-lite-bpi-r4-5.4-sd-emmc.img.zip debian-11-bullseye-lite-bpi-r4-5.4-sd-emmc.img | sudo dd bs=16M of=/dev/disk/by-id/usb-YOU-DRIVE
    ```
16. Eject the USB drive from your PC safely and connect it to BPI-R4
17. Start the USB bus on BPI-R4 with command `usb start` and make sure your USB drive is recognized
    ```
    MT7988> usb start
    starting USB...
    Bus xhci@11200000: xhci-mtk xhci@11200000: hcd: 0x0000000011200000, ippc: 0x0000000011203e00
    xhci-mtk xhci@11200000: ports disabled mask: u3p-0x0, u2p-0x0
    xhci-mtk xhci@11200000: u2p:1, u3p:1
    Register 200010f NbrPorts 2
    Starting the controller
    USB XHCI 1.10
    scanning bus xhci@11200000 for devices... 5 USB Device(s) found
           scanning usb for storage devices... 1 Storage Device(s) found
    ```
18. (Optional) Make sure the USB drive contains the correct system image by `ls usb 0:5` and `ls usb 0:6`; when in doubt, disconnect, rewrite the image, reconnect and run `usb reset`
    ```
    MT7988> ls usb 0:5
                bananapi/

    0 file(s), 1 dir(s)

    MT7988> ls usb 0:6
    <DIR>       4096 .
    <DIR>       4096 ..
    <SYM>          7 bin
    <DIR>       4096 boot
    <DIR>       4096 dev
    <DIR>       4096 etc
    <DIR>       4096 home
    <SYM>          7 lib
    <DIR>       4096 lost+found
    <DIR>       4096 media
    <DIR>       4096 mnt
    <DIR>       4096 opt
    <DIR>       4096 proc
    <DIR>       4096 root
    <DIR>       4096 run
    <SYM>          8 sbin
    <DIR>       4096 srv
    <DIR>       4096 sys
    <DIR>       4096 tmp
    <DIR>       4096 usr
    <DIR>       4096 var
    ```
19. Type the following commands to boot from USB (as serial is prone to corruption, it's recommended to type manually or copy-paste line-by-line, don't paste all lines in one go as that would usually result in character drop)
    ```
    load usb 0:5 0x47000000 bananapi/bpi-r4/linux-5.4/dtb/mt7988a-bananapi-bpi-r4-emmc-snand.dtb
    load usb 0:5 0x46000000 bananapi/bpi-r4/linux-5.4/uImage
    setenv bootargs console=ttyS0,115200n8 root=/dev/sda6 rw
    bootm 0x46000000 - 0x47000000
    ```
20. Login via console with user name `root` password `bananapi`
21. Run `lsblk` to check for partitions, the result shall looks like follows:
    ```
    root@BPI-R4:~# lsblk
    NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    sda             8:0    0 931.5G  0 disk 
    ├─sda1          8:1    0     4M  0 part 
    ├─sda2          8:2    0   512K  0 part 
    ├─sda3          8:3    0     2M  0 part 
    ├─sda4          8:4    0     4M  0 part 
    ├─sda5          8:5    0   256M  0 part /boot
    └─sda6          8:6    0   6.8G  0 part /
    mtdblock0      31:0    0   128M  0 disk 
    mtdblock1      31:1    0     1M  1 disk 
    mtdblock2      31:2    0   512K  0 disk 
    mtdblock3      31:3    0     4M  0 disk 
    mtdblock4      31:4    0     2M  0 disk 
    mtdblock5      31:5    0 112.5M  0 disk 
    mmcblk0       179:0    0   7.3G  0 disk 
    ├─mmcblk0p1   179:1    0   512K  0 part 
    ├─mmcblk0p2   179:2    0     2M  0 part 
    ├─mmcblk0p3   179:3    0     4M  0 part 
    ├─mmcblk0p4   179:4    0    32M  0 part 
    ├─mmcblk0p5   179:5    0   448M  0 part 
    └─mmcblk0p128 259:1    0     4M  0 part 
    mmcblk0boot0  179:8    0     4M  1 disk 
    mmcblk0boot1  179:16   0     4M  1 disk
    ```
    Here `mmcblk0` is partitioned eMMC with mainline OpenWRT image, and `sda` is our live USB
22. Re-partition eMMC, delete `mmcblk0p4`, `mmcblk0p5` and create a single, big root partition to take all remaining space (no need for seperate `/boot`)
    ```
    root@BPI-R4:~# fdisk /dev/mmcblk0

    Welcome to fdisk (util-linux 2.36.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before us[  127.124908] Alternate GPT is invalid, using primary GPT.
    ing the write co[  127.131090]  mmcblk0: p1 p2 p3 p4 p5 p128
    mmand.

    GPT PMBR size mismatch (1048607 != 15269887) will be corrected by write.
    The backup GPT table is corrupt, but the primary appears OK, so that will be used.
    The backup GPT table is not on the end of the device. This problem will be corrected by write.

    Command (m for help): d
    Partition number (1-5,128, default 128): 5

    Partition 5 has been deleted.

    Command (m for help): d
    Partition number (1-4,128, default 128): 4

    Partition 4 has been deleted.

    Command (m for help): n
    Partition number (4-127, default 4): 
    First sector (21504-15269854, default 22528): 
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (22528-15269854, default 15269854): 

    Created a new partition 4 of type 'Linux filesystem' and of size 7.3 GiB.

    Command (m for help): t
    Partition number (1-4,128, default 128): 4
    Partition type or alias (type L to list all): 25

    Changed type of partition 'Linux filesystem' to 'Linux root (ARM-64)'.

    Command (m for help): w
    The partition table has been altered.
    Calling i[  138.521811]  mmcblk0: p1 p2 p3 p4 p128
    octl() to re-read partition table.
    Syncing disks.
    ```
23. Create and ext4 filesystem on the new, big root on eMMC
    ```
    mkfs.ext4 -m 0 /dev/mmcblk0p4
    ```
24. Mount the newly created ext4 filesystem
    ```
    mount /dev/mmcblk0p4 /mnt
    ```
25. Clone the whole running system to eMMC
    ```
    rsync -qaAXH --exclude='/dev/*' --exclude='/proc/*' --exclude='/sys/*' --exclude='/tmp/*' --exclude='/run/*' --exclude='/mnt/*' --exclude='/media/*' --exclude='/lost+found/' / /mnt
    ```
26. Replace the whole content of eMMC fstab
    ```
    echo '/dev/mmcblk0p4 / ext4 rw 0 1' > /mnt/etc/fstab
    ```
27. Power off BPI-R4 after syncing all writes
    ```
    umount /mnt
    poweroff
    ```
28. Disconnect the USB drive from BPI-R4
29. Toggle BPI-R4's boot switch to 1,0 (left down, right up) for eMMC
30. Power on, hold down arrow and wait for the U-Boot menu to show up and the auto-selection is interrupted. Make sure the menu starts with ( ( ( OpenWrt ) ) ) [eMMC]
31. Set boot environment
    ```
    setenv boot_default 'load mmc 0:4 0x47000000 boot/bananapi/bpi-r4/linux-5.4/dtb/mt7988a-bananapi-bpi-r4-emmc-snand.dtb; load mmc 0:4 0x46000000 boot/bananapi/bpi-r4/linux-5.4/uImage; setenv bootargs console=ttyS0,115200n8 root=/dev/mmcblk0p4 rw; bootm 0x46000000 - 0x47000000'
    saveenv
    ```
32. Try to boot into the system on eMMC
    ```
    run boot_target
    ```
33. After booting, `lsblk` to confirm the current boot device
    ```
    root@BPI-R4:~# lsblk
    NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    mtdblock0      31:0    0   128M  0 disk 
    mtdblock1      31:1    0     1M  1 disk 
    mtdblock2      31:2    0   512K  0 disk 
    mtdblock3      31:3    0     4M  0 disk 
    mtdblock4      31:4    0     2M  0 disk 
    mtdblock5      31:5    0 112.5M  0 disk 
    mmcblk0       179:0    0   7.3G  0 disk 
    ├─mmcblk0p1   179:1    0   512K  0 part 
    ├─mmcblk0p2   179:2    0     2M  0 part 
    ├─mmcblk0p3   179:3    0     4M  0 part 
    ├─mmcblk0p4   179:4    0   7.3G  0 part /
    └─mmcblk0p128 259:1    0     4M  0 part 
    mmcblk0boot0  179:8    0     4M  1 disk 
    mmcblk0boot1  179:16   0     4M  1 disk
    ```
34. Reboot to confirm the boot environment modification is persistent.
