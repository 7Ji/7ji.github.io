---
layout: post
title:  "Stay low! Make the Windows bootloader behave in a dual boot environment"
date:   2023-04-18 15:00:00 +0800
categories: booting
---

If you're running a dual-boot Windows + Linux setup on a modern x86-64 platform with normal UEFI support, chances are you will find the `Windows Boot Manager` put itself at the first UEFI boot entry proudly every time you boot into it, this gets very annoying because you usually don't want to use it for the most of the time, and getting into BIOS or using efibootmgr to fix your boot order is very tedious.

For example if I have these UEFI boot entries in a carefully ordered way so I can boot into my Arch installation immediately at `Boot0000` without any overhead or use `systemd-boot` at `Boot0001` to choose alternative kernels or Windows:

```
> efibootmgr -u
BootCurrent: 0000
Timeout: 2 seconds
BootOrder: 0000,0001,0002
Boot0000* Arch Linux (ZEN kernel)       HD(1,GPT,4f81ae74-5b01-e84d-a044-321dcee1bc18,0x100,0x10000)/File(\vmlinuz-linux-zen)initrd=\amd-ucode.img initrd=\initramfs-linux-zen.img root=UUID=d1c93fc2-99cc-4221-b6ae-bd3885979849 rw nvidia_drm.modeset=1
Boot0001* Linux Boot Manager    HD(1,GPT,4f81ae74-5b01-e84d-a044-321dcee1bc18,0x800,0x80000)/File(\EFI\systemd\systemd-bootx64.efi)
Boot0002* Windows Boot Manager     HD(1,GPT,8d41800c-75c4-45b1-8fef-6a83ea7a8c3f,0x100,0x6400)/File(\EFI\Microsoft\Boot\bootmgfw.efi)
```

Then after a boot into Windows, the `BootOrder` will be overwritten to `0002,0000,0001`, with Windows proudly putting itself to the first

Just deleting the `Windows Boot Manager` entry with `efibootmgr -b 2 -B` does not work, Windows will re-add it back to `fix things up`

Thankfully Windows will only try to re-add the entry if and only if it could find a valid boot manager at the predefined location `\EFI\Microsoft\Boot\bootmgfw.efi`, so this could be easily worked around by mounting the ESP partition for Windows (if on different disk and has dedicated ESP), or just edit your ESP partition (if on same disk):

```
sudo mount -o noatime /dev/[TheWindowsESP] /mnt
```

And then rename the `Microsoft` folder 
```
cd /mnt/EFI
sudo mv Microsoft Micro$oft
```

And, if Windows is on its dedicated disk, or it has installed itself as a fallback bootloader over the Linux one on the same disk, delete the fallback `BOOT` folder
```
sudo rm -rf BOOT
```

The Windows boot entry can be deleted, so you can add it to your Linux boot loader.
```
sudo efibootmgr -b [TheWindowsBootNum] -B
```

This is an example boot entry for `systemd-boot`:

 - `/boot/loader/entries/windows.conf`
    ```
    title   Windows
    efi     /Shell.efi
    options -nointerrupt -noconsolein -noconsoleout windows.nsh
    ```
 - `/boot/windows.nsh`:
    ```
    HD1b:EFI\Micro$oft\Boot\bootmgfw.efi
    ```
    The FS alias `HD1b` could be found by loading an EFI shell directly and run a `map` command, an example EFI shell boot entry
    - `/boot/loader/entries/shell.conf`
        ```
        title UEFI SHELL
        efi     /Shell.efi
        ```
 - `Shell.efi` is copied from `/usr/share/edk2-shell/x64/Shell.efi` provided by package `edk2-shell`

Reboot and try out, Windows can be booted by selecting the newly added boot entry from your Linux boot manager, but it can't add nor adjust the EFI boot entryies nor orders.

And, if you still want to boot into Windows directly from your BIOS boot menu, you could add it back as a single EFI boot entry:

```
sudo efibootmgr --create --disk [TheWindowsDisk] --part 1 --label 'Micro$oft Window$' --loader '\EFI\Micro$oft\Boot\bootmgfw.efi'
```
Don't forget to re-order the entries as the newly added one will always be set to first
```
sudo efibootmgr -o 0,1,2
```
This will give you an boot entry like the following, which will not put itself to first even if you boot into it from your BIOS boot menu
```
Boot0002* Micro$oft Window$     HD(1,GPT,8d41800c-75c4-45b1-8fef-6a83ea7a8c3f,0x100,0x6400)/File(\EFI\Micro$oft\Boot\bootmgfw.efi)
```