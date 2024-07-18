---
layout: post
title:  "Firmware Image Package (bootloader image) hacking on Amlogic"
date:   2024-07-17 12:00:00 +0800
categories: booting
---

## Background

### FIP introduction

Most people call the data chunk that makes an Amlogic device bootable a "bootloader", or "u-boot", but that's far from the truth. The data chunk is actually a Firmware Image Package, or simply FIP. And U-Boot is only part of it (BL33).

As per ARM Trusted Firmware Design, FIP is an archive that contains multiple parts of Trusted Firmware-A (TF-A)

> Using a Firmware Image Package (FIP) allows for packing bootloader images (and potentially other payloads) into a single archive that can be loaded by TF-A from non-volatile platform storage. A driver to load images from a FIP has been added to the storage layer and allows a package to be read from supported platform storage.

A modern ARM device must has FIP on its persistent storage so it could boot. Basically SoCs embedds only a small chunk of code (BL1/BootROM) that's responsible for early SoC initialization and loading FIP. All later logic happens in different parts of FIP (BL2, BL2, BL30, etc).

### Amlogic FIP storage

On Amlogic platform, the FIP is usually stored on the embedded MMC (eMMC) device, or NAND. It could also be stored on SD card as it's pretty much the same thing as eMMC user partition.

For simplicity, let's only focus on the eMMC scenarios.

#### eMMC hardware partition

All eMMC devices have multiple hardware partitions: two boot hwpartitions that're usually 4MiB in size and read-only, one RPMB (Replay Protected Memory Block) hwpartition that's used for safe data transfer between safe and unsafe worlds, and lastly one user hwpartition that's pretty much the same as a SD card.

The partition layout is usually like the following:

|name|offset|size|linux dev e.g.|
|-|-|-|-|
|boot0|0|4MiB|mmcblk1boot0 (block)|
|boot1|4MiB|4MiB|mmcblk1boot1 (block)|
|rpmb|8MiB|4MiB|mmcblk1rpmb (char)|
|user|12MiB|-|mmcblk1 (block)|

User-defined partitions (MBR, GPT, whatever they are) only live in user hardware partition and Linux kernel treats all four hardware partitions as if they're four different devices.

#### FIP partition

Before GXL family, until GXBB family, i.e. all SoCs until S905 no suffix, FIP could only be stored on eMMC user partition, even though there are two boot partitions. Since GXL family, i.e. all SoCs since S905X/D/W/L, FIP could also be stored on eMMC boot partitions.

The following chart describes the boot flow on different eMMC hareware partitions on different Amlogic SoCs

|target|<= GXBB (S905)|>= GXL (s905X/D/W/L)|
|-|-|-|
|user|boot|boot 1|
|boot0|skip|boot 2|
|boot1|skip|boot 3|

Whichever the partition is, FIP image is always stored with a 512 byte offset, to avoid conflicting with MBR on eMMC user partition (even when it's stored on boot partition).

## FIP dumping

### from Amlogic USB burning image

Amlogic USB burning image, usually with a `.img` suffix, is Amlogic's proprietary format that's used by their Amlogic USB Bunring Tool to burn system images onto devices with Amlogic SoCs.

You can use my [ampack] tool to unpack such image. The FIP images would be dumped along other partitions as `bootloader.PARTITION`

```
ampack unpack mibox3.img mibox3-unpacked
```

### from eMMC user partition

This is usually the place on a running device that you would want to dump FIP from.

Be sure you dumped from the right block device (the one with boot0 and boot1 companions) and with correct offset (512 bytes). The maximum data you would want to get is usually 4MiB - 512B.

#### On Android with root permission

Enable ADB network debugging and get a shell, then run `su` to gain root permission:
```
adb connect [IP address]
adb shell
su
```

Dump the FIP on device and exit
```
dd if=/dev/mmcblk1 of=/sdcard/fip bs=512 skip=1 count=8191
exit
exit
```

Then pull the fip image
```
adb pull /sdcard/fip fip
```

#### On generic Linux distro

For generic Linux scenarios dd directly without adb pulling.
```
dd if=/dev/mmcblk1 of=fip bs=512 skip=1 count=8191
```

### from eMMC boot partitions

This is a speical case, as eMMC boot partitions are never used by any stock Amlogic Android image or third party Linux images for Amlogic devices. If this is the case (like on my tinkered box), dump from the corresponding `boot0` or `boot1` block device instead.
```
dd if=/dev/mmcblk1boot0 of=fip.img bs=512 skip=1
```

## FIP tinkering

### FIPs for SoCs since GXL

For SoCs since GXL, all parts in a FIP are signed and encrypted, and then the FIP image is signed and encrypted as a whole. It would be in Amlogic's proprietary format so you need a dedicated [gxlimg] tool to unsign/decrypt/encrypt/sign.

#### FIP decryption

Use [gxlimg] to decrypt and extract all BL parts from FIP

```
mkdir fip-parts
gxlimg --extract unpacked/bootloader.PARTITION fip-parts
```

E.g. for R3300L, the output folder would contain the following members:

```
> ls fip-parts/
bl2.sign  bl301.enc  bl30.enc  bl31.enc  bl33.enc  fip  fip.enc
```

#### Bl33 (u-boot) encryption

Use the [gxlimg] tool to encrypt your own BL33 image before substitute the original BL33, assuming `../u-boot.bin` is your own u-boot binary

```
gxlimg --type bl3x ../u-boot.bin bl33.enc
```

#### FIP repack

Use the [gxlimg] tool to pack everything (your new `bl33.enc` and all other original parts) into a new FIP image

```
gxlimg --type fip --bl2 bl2.sign --bl30 bl30.enc --bl301 bl301.enc --bl31 bl31.enc --bl33 bl33.enc ../new-fip.img
```

### FIPs for SoCs until GXBB

A FIP image for SoCs since GXL is signed and encrypted as a whole, all parts are stored plainly inside it. It would be in slightly modified TF-A's reference FIP format when not encrypted. You only need the [meson-tools] to unsign/sign the FIP.

#### FIP unsigning

Use `unamlbootsig` from [meson-tools] to unsign the FIP image

```
unamlbootsig original original.dec
```

This would also hint on the offset and size of different parts.

#### Bl33 (u-boot) substitution

Use [my script](https://github.com/7Ji/u-boot/blob/be245d6842da62c31a5c96f9041b3f414b0fe57d/gxbb_p201_xiaomi_mdz-16-aa/new_fip.py) to replace BL33 inside FIP image, it assumes the follows:
- unsigned BL33 is at `original.dec`
- a folder `parts` exist
- new `u-boot.bin` is at `../../u-boot/u-boot.bin`
- BL2 size is `0xC000`

It works without modification on mibox3, but you'll need to modify the script accordingly for other devices.

```
mkdir parts
python new_fip.py
```

The substituted, unsigned new FIP image would be `with_mainline`

#### FIP signing

Use `amlbootsig` from [meson-tools] to sign the FIP image
```
amlbootsig with_mainline with_mainline.enc
```


[gxlimg]: https://github.com/repk/gxlimg
[meson-tools]: https://github.com/afaerber/meson-tools

## FIP writing

### Direct write

You can write the new FIP image to either one of the following places:
- eMMC user partition (mmcblkN)
- eMMC boot partition 1 (mmcblkNboot0)
- eMMC boot partition 2 (mmcblkNboot1)

If you write it to later ones then be sure the earlier ones don't have valid FIP on them (e.g. if you write to boot partition 2, then be sure to erase boot partition 1 and first 4M of user partition)

If you're writing FIP image prepared using the methods here then remember to leave a 512 offset (reserved for MBR, yes, even for boot partitions).

E.g. to write to eMMC user hwpartition:
```
sudo dd if=new-fip.img of=/dev/mmcblkN bs=512 seek=1
```

E.g. to write to eMMC boot hwpartition 1:
```
echo 0 | sudo tee /sys/block/mmcblkNboot0/force_ro 
sudo dd if=new-fip.img of=/dev/mmcblkNboot0 bs=512 seek=1
```

If you're writing `u-boot.bin.sd.bin` built with `amlogic-boot-fip` then there's no need for the 512 offset, but remember to restore the MBR partitions (`sfdisk -d` then `sfdisk`) you had on eMMC user partition.

### Fake Burning image

For SoCs until GXBB family, it's possible to replace the `bootloader.PARTITION` part in the burning image, remove unneeded Android parts, and repack it up. 

E.g. for mibox3, only the following parts are needed to get an Amlogic USB Burning Image that boots the box into a state where booting mainline kernel from USB is possible.

```
> ls unpacked
aml_sdc_burn.ini  bootloader.PARTITION  DDR.USB  meson1.dtb  platform.conf  UBOOT.USB
```

using [ampack] to pack the above parts into an Amlogic USB Burning Image
```
ampack pack mibox3-unpacked mibox3-bare-but-mainline.img
```

[ampack]: https://github.com/7Ji/ampack