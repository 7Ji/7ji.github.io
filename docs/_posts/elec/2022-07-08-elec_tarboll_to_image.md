---
layout: post
title:  "Make a bootable CoreELEC/EmuELEC drive with update tarboll instead of image"
date:   2022-07-08 17:17:12 +0800
categories: elec
---
## Introduction
This documents a method to make a bootable CoreELEC/EmuELEC drive, with update tarboll (.tar) instead of images (.img/.img.gz), this will be of good use if you want to use the nightly EmuELEC builds available at [my EE nightly build site](https://ee.fuckblizzard.com) directly as I don't have enough disk space to store images so there's only update tarbolls there

You get the following benifits to prepare a bootable CE/EE disk in this way
* You can decide the partition size by yourself (So there won't be ~300M or ~1G "wasted" )
* Save the time you need for fs-resize during first boot (as we already partition the whole disk by ourselves)
* You can preload any configs/roms as you like

The disadvantage though:
* Don't expect support from Team CE nor Team EE, as this is not the use case supported by them


## Preperation

You will need the following tools and environments:
* A Linux machine, physical or virtual, with USB/SD access to the drive you're going to use for the bootable disk. If you're using Windows then VirtualBox+USB passthrough should work, but I suggest a physical Linux machine.
* Root access on that Linux machine, as root or with sudo
* 2 * the size of the update tarboll on that machine


And as always:
* You are familiar with CLI operations under Linux environment

## Operation
**Don't copy/paste the commands blindly, most of them will differ depending on your actual drive and the tarboll you use and the image you want to create!**

Download the tarboll you'll use
````
wget https://ee.fuckblizzard.com/EmuELEC-Amlogic-ng.aarch64-4.5-Nexus_nightly_20220706.tar
````
Create temporary folders for the tarboll, the part 1 and the part 2
````
mkdir {tarboll,part1}
````
Extract the content of that tarboll
````
tar --strip-components=1 -C tarboll -xvf EmuELEC-Amlogic-ng.aarch64-4.5-Nexus_nightly_20220706.tar
````
Move/Copy kernel and system images
````
mv tarboll/target/KERNEL part1/kernel.img
mv tarboll/target/SYSTEM part1/
````
Move/Copy the dtb
````
mv tarboll/3rdparty/bootloader/device_trees/your_dtb.dtb part1/dtb.img
````
Move/Copy the bootloader specific files (**Multiple** steps are needed to be performed depending on your device)
- All devices that need specialized images (Odroid N2, Odroid C2, etc), **and** all generic Amlogic-ng devices
  ````
  mv tarboll/3rdparty/bootloader/config.ini part1/
  ````
- All devices that need specialized images
  ````
  mv tarboll/3rdparty/bootloader/**[DEVICE_NAME]**_boot.ini part1/boot.ini
  mv tarboll/3rdparty/bootloader/**[VENDOR_NAME]**-boot-logo-1080.bmp.gz part1/boot-logo-1080.bmp.gz
  ````
- LibreComputer devices
  ````
  mv tarboll/3rdparty/bootloader/**[DEVICE_NAME]**_chain_u-boot part1/u-boot.bin
  mv tarboll/3rdparty/bootloader/libretech_chain_boot part1/boot.scr
  ````
- Generic Amlogic (both -ng and -old/non-ng) devices
  ````
  mv tarboll/3rdparty/bootloader/aml_autoscript part1/
  ````
- Generic Amlogic-ng devices
  ````
  mv tarboll/3rdparty/bootloader/Generic_cfgload part1/cfgload
  ````

A summary table

| Device                | Generic Amlogic-ng | Generic Amlogic-old | LibreComputer | Other specialized |
|-----------------------|--------------------|---------------------|---------------|-------------------|
| aml_autoscript        | √                  | √                   |               |                   |
| boot.ini              |                    |                     | √             | √                 |
| boot.src              |                    |                     | √             |                   |
| boot-logo-1080.bmp.gz |                    |                     | √             | √                 |
| cfgload               | √                  |                     |               |                   |
| config.ini            | √                  |                     | √             | √                 |
| dtb.img               | √                  | √                   | √             | √                 |
| kernel.img            | √                  | √                   | √             | √                 |
| SYSTEM                | √                  | √                   | √             | √                 |
| u-boot.bin            |                    |                     | √             |                   |


Get the size of files for part1
````
du -sh part1
````
For me it's 1013M, that means the minumum size for the part1 on disk will be 1013M+3M=1016M, you can go lower as the actual overhead of FAT32 is sometimes confusing, or maybe you'll need to go higher for the same reason. You can play it safe with a 2G size anyway.

Partition the disk, create a new MSDOS partition table, a part 1 with type "W95 FAT32", a part 2 with default "linux" type. 
````
fdisk /dev/yourDrive
````

**If you're preparing the image for devices that need specialized image, you need to leave 4M before the first partition, as this area will be used to store the u-boot for these devices, using fdisk you would need to create the first partition starting from sector 8192 instead of the default 4096**

The following is the log when I create the needed partition table for generic Amlogic-old device, since it does not need u-boot I start at the default sector 4096 (4096 sectors * 512 bytes/sector = 2097152 bytes = 2 MiB, and 8192 sectors then means 4 MiB) 
````
[root@Lab7Ji nightly_0706]# fdisk /dev/sde

Welcome to fdisk (util-linux 2.38).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): o
Created a new DOS disklabel with disk identifier 0xb2d8b4fb.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (1-4, default 1): 
First sector (2048-61849599, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-61849599, default 61849599): +1016M

Created a new partition 1 of type 'Linux' and of size 1016 MiB.
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 0b
Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (2-4, default 2): 
First sector (2082816-61849599, default 2082816): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2082816-61849599, default 61849599): 

Created a new partition 2 of type 'Linux' and of size 28.5 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
````
Create a FAT32 file system on the first partition
````
mkfs.vfat -F 32 -n EMUELEC /dev/sde1
````
Create a temporary directory for the partition
````
mkdir /mnt/eep1
````
Mount it
````
mount -o rw /dev/sde1 /mnt/eep1
````
Copy all files to part1
````
cp -rva part1/* /mnt/eep1/
````
if the disk space is not enough, you'll need to go back and re-partition the disk

Create a ext4 partition on the second partition
````
mkfs.ext4 -m 0 -L STORAGE /dev/sde2
````
Optionally mount it and populate it with your roms/configs/etc

An additional ``-d`` argument will be of use if you want to populate it with a local folder, without even mounting it, remember to run mkfs.ext4 with fakeroot though if you want to do so

If your device uses ``boot.ini`` (devices that need specialized image) 
  - Get partition UUIDs
    ````
    blkid /dev/sde1
    blkid /dev/sde2
    ````
    For me it's ``47C2-F67F`` and ``ced8cd7c-3b99-4644-82dd-167ae9a45a79``
  - Set partition UUID in ``boot.ini``

    Find line
    ````
    setenv rootopt "BOOT_IMAGE=kernel.img boot=UUID=@BOOT_UUID@ disk=UUID=@DISK_UUID@"
    ````
    Change it according to your actual UUIDs like
    ````
    setenv rootopt "BOOT_IMAGE=kernel.img boot=UUID=47C2-F67F disk=UUID=ced8cd7c-3b99-4644-82dd-167ae9a45a79"
    ````
  
Optionally pre-populate the /usr/config units, if the SD controller on your device is slower than the one you are using on your Linux machine (This is only useful for EmuELEC since it needs to populate ~480M for the first boot, for CoreELEC there's not so much data)
- Create temporary folders and mount both the  ``SYSTEM`` image
  ````
  mkdir -p /mnt/eesys
  mkdir -p /mnt/eep2
  mount -o ro part1/SYSTEM /mnt/eesys
  mount -o rw,noatime /dev/sde2 /mnt/eep2
  ````
  Notice that you may need to set up loop devices if your Linux machine does not support mounting squashfs image directly (loop device can be automatically set up)

  This will be needed before the above commands for some old Linux systems
  ````
  losetup -f part1/SYSTEM
  ````
  And remember to replace part1/SYSTEM in the mount command to the used loop device (in my case it's /dev/loop0, but on Ubuntu since it's bloated with snapd, many loop devices will be used and it might be even /dev/loop9)
- Populate configs
  ````
  mkdir /mnt/eep2/.config
  cp -rva /mnt/eesys/usr/config/* /mnt/eep2/.config/
  ````

Umount all mounted partitions
````
umount /dev/sde?
umount /dev/ee*
````
(If used) Detach all unused loop devices
````
losetup -D
````
For devices that need u-boot, i.e. those need specialized images (Odroid N2, etc), remember we leave a 4M gap before the first partition? We'll write the u-boot to the disk now
- Remember to replace sde with correct drive
  ````
  dd if=tarboll/3rdparty/bootloader/CorrespondingUboot of=/dev/sde conv=fsync,notrunc bs=1 count=112
  dd if=tarboll/3rdparty/bootloader/CorrespondingUboot of=/dev/sde conv=fsync,notrunc bs=512 skip=1 seek=1
  ````
  Notice we skip the bytes 113-512, this is where the partition table is stored


Sync all disk I/O
````
sync
````

Optionally notify kernel to remove the drive
````
echo 1 > /sys/block/sde/device/delete 
````
Remove the drive, and plug it on your device. Then you can start to use it as how you normally boot CE/EE for the first time, except you 