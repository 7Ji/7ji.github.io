---
layout: post
title:  "Gotchas when booting from virtiofs root"
date:   2025-05-23 18:00:00 +0800
categories: booting
---

virtiofs is a nice host-passthrough "fs" with which a virtual machine can use the host fs almost directly, saving the overhead of "guest fs -> guest disk -> host file -> host fs" hassle, a virtiofs root is especially useful when access to files in guest root fs from host directly is needed, e.g. when doing frequent debugging, or vice versa.

It might seem enough to just set the kernel cmdline to:
```sh
root=root rootfstype=virtiofs
```

However to make a virtual machine actually boot from such root you need to go through a few gotchas:

## Common
- In libvirt, you need to allocate a `dir` type storage pool and then `mkdir` a root-owned subfolder inside the pool, the pool could be reused while the subfolder needs to be VM-specific, the corresponding storage config file looks like this:
    ```sh
    > cat /etc/libvirt/storage/filesystems.xml
    ```
    ```xml
    <pool type='dir'>
    <name>filesystems</name>
    <uuid>b36d710f-7a61-49f7-935b-061b12f24c8a</uuid>
    <capacity unit='bytes'>0</capacity>
    <allocation unit='bytes'>0</allocation>
    <available unit='bytes'>0</available>
    <source>
    </source>
    <target>
        <path>/var/lib/libvirt/filesystems</path>
    </target>
    </pool>
    ```
    The subfolder:
    ```sh
    > ls -ldh /var/lib/libvirt/filesystems/root.debian12
    drwxr-xr-x 18 root root 4.0K May 23 17:55 /var/lib/libvirt/filesystems/root.debian12/
    ```
- The guest needs to have a "filesystem" "device" that points to the subfolder, i.e. the guest config shall have the following snippet:
    ```sh
    > sed -n '/<filesystem/,/<\/filesystem/p' /etc/libvirt/qemu/debian12.xml
    ```
    ```xml
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs'/>
      <source dir='/var/lib/libvirt/filesystems/root.debian12'/>
      <target dir='root'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </filesystem>
    ```
- The guest fstab needs only the following line then:
    ```yaml
    root / virtiofs defaults 0 0
    ```
- If you need to use overlayfs with upperdir pointing to path inside the virtual root, the `virtiofsd` needs addtional arguments to allow xattr and keep `cap_sysadmin`:
    ```xml
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs' queue='1024'/>
      <binary path='/usr/lib/virtiofsd+sys_admin' xattr='on'/>
      <source dir='/var/lib/libvirt/filesystems/root.arb-x64-builder'/>
      <target dir='root'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </filesystem>
    ```
    ```sh
    > cat /usr/lib/virtiofsd+sys_admin
    ```
    ```sh
    #!/bin/sh
    exec /usr/lib/virtiofsd -o modcaps=+sys_admin "$@"
    ```
- If you want to share something between VMs it's not recommended to re-use the root subfoldder (which is as bad as mounting a virtual disk that is being used), just create another shared subfolder.

## Debian guest

- You must either install a system to a virtual disk first then extract the files, or debootstrap manually
- If the system rootfs folder was extracted from a locally installed disk image, then `/etc/initramfs-tools/conf.d/resume` must be purged before a rerun of `update-initramfs -u`, otherwise the initrd would try to resume from the missing block device and break
- You must include `virtiofs` in `/etc/initramfs-tools/modules` before a rerun of `update-initramfs -u` if you're preparing the initrd from a running system on virtual disk, otherwise the `virtiofs` module would not be included. (However, if you've already made it into a system running from virtiofs root and are optimizing the initrd, or are preparing the initrd from a virtiofs chroot, this could be omitted)
- A run of `update-initramfs -u` is certainly needed after the above gotchas sorted out.
- The guest kernel needs to be booted directly by hypervisor, without a bootloader, and it's recommended to use the `vmlinuz` and `initrd.img` symlinks instead of the real files:

    ```sh
    > sed -n '/<os/,/<\/os/p' /etc/libvirt/qemu/debian12.xml
    ```
    ```xml
    <os>
        <type arch='x86_64' machine='pc-q35-10.0'>hvm</type>
        <kernel>/var/lib/libvirt/filesystems/root.debian12/vmlinuz</kernel>
        <initrd>/var/lib/libvirt/filesystems/root.debian12/initrd.img</initrd>
        <cmdline>root=root rootfstype=virtiofs</cmdline>
        <boot dev='hd'/>
        <bootmenu enable='no'/>
    </os>
    ```
- If you've prepared the root from a virtual disk bootable image, grub stuffs needs to be purged later:
```
apt purge --autoremove grub2 grub-common
rm -rf /boot/grub
```

## Arch Linux guest
- Only `mkinitcpio` and `dracut` initrd makers support booting from `virtiofs` natively, `booster` needs [my patchset](https://github.com/anatol/booster/pull/298), but `dracut` is not recommended as its initramfs paths are undetermined containing kernel version (see below).
- The target rootfs can be prepared by either `pacstrapping` in host using `arch-install-scripts` or `pacstrapping` in target using archiso, it's not recommended to extract the rootfs from a installed system that runs on either virtual disk or physical machine
- The guest kernel needs to be booted directly by hypervisor, without a bootloader, the kernel path is determined and the initrd path also unless you're using `dracut` (so better use `booster` or `mkinitcpio`)

    ```sh
    > sed -n '/<os/,/<\/os/p' /etc/libvirt/qemu/archlinux.xml
    ```
    ```xml
    <os>
        <type arch='x86_64' machine='pc-q35-10.0'>hvm</type>
        <kernel>/var/lib/libvirt/filesystems/root.archlinux/boot/vmlinuz-linux</kernel>
        <initrd>/var/lib/libvirt/filesystems/root.archlinux/boot/booster-linux.img</initrd>
        <cmdline>root=root rootfstype=virtiofs</cmdline>
        <boot dev='hd'/>
        <bootmenu enable='no'/>
    </os>
    ```
