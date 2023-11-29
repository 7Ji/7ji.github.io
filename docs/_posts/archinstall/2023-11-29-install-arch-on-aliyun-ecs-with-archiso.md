---
layout: post
title:  "Installing ArchLinux on Aliyun ECS with ArchISO"
date:   2023-11-29 19:50:00 +0800
categories: archinstall
---

The ArchISO is a pretty powerful tool that helps you to bootstrap an ArchLinux installation from ground up. In this blog I'll document how I used that to install an ArchLinux installation onto an Aliyun ECS, with custom partitioning and bootup configuration.

This should also apply to any other VPS provider that supports VNC. But the actual result is not guaranteed

Also, as Arch has [its own Installation Guide](https://wiki.archlinux.org/title/Installation_guide#Verify_signature), I'll only focus on the parts that differ quite a bit from the generic installation process.

## Prerequisite
This is all done on an `ecs.e-c1m1.large` instance, with a spec of 2C2C. It came with Aliyun's Debian 12.2 pre-installed, and is booted in legacy BIOS mode with GRUB2 as its bootloader. The most important spec here is that:
- The RAM must be larger than the size of the squashfs image inside ArchISO, which usually takes a space of less than 700MiB. We'll need to load the system image into RAM so we can freely re-partition the disk. The squashfs image inside the 2023-11 release took 685.2MiB, so I guess the bare minimum is 1GiB.
- The bootloader is configurable and allows to boot from a loopback image. GRUB2 suits that. AFAIK syslinux also supports that but since most VPS providers just use the default GRUB2 I'll only focus on GRUB2.
- A readable partition should exist for GRUB that could be used to store the ISO file. Here it is the root partition with an ext4 fs, `/dev/vda3`

## Preparing the ISO
It's pretty easy to find the latest release of ArchISO and a list of mirror sites from [Arch Linux Downloads](https://archlinux.org/download/). What's better is that Aliyun has their own mirror for the whole Arch repo, so you can just download from their site:
```
wget https://mirrors.aliyun.com/archlinux/iso/latest/archlinux-x86_64.iso
```

Don't forget to verify its signature as documented in [Arch Installation guide](https://wiki.archlinux.org/title/Installation_guide#Verify_signature)

Use `blkid` to get the label of the image and record that elsewhere which would be used later
```
blkid archlinux-x86_64.iso
```
As of writing the latest release has a label `ARCH_202311`, currently it's named in the format of `ARCH_[YYYYMM]` but don't rely on the naming style too much and always check that by yourself.

You can then mv the ISO to filesystem root for an easier to lookup path
```
sudo mv archlinux-x86_64.iso /archiso
```
Since we're going to erase the old partitions anyway, tainting the root fs is not that a big deal in my opinion.

## Modifying GRUB config
Use a text editor to modify `/boot/grub/grub.cfg`
```
sudo vim /boot/grub/grub.cfg
```

Append a boot menu entry before the line like the following:
```
menuentry "ArchISO (x86_64)" {
    insmod iso9660
    set isofile=/archiso
    loopback lo0 $isofile
    linux (lo0)/arch/boot/x86_64/vmlinuz-linux archisolabel=ARCH_202311 img_dev=/dev/vda3 img_loop=$isofile copytoram=y
    initrd (lo0)/arch/boot/x86_64/initramfs-linux.img
}
```
This should be after all other menu entries and before the following line:
```
### END /etc/grub.d/10_linux ###
```

Note that:
- The `isofile` should be a path relative to the filesystem root
- The `archisolabel` should be what you recorded in the previous step
- The `img_dev` should be the partition storing your existing rootfs.
- **DO NOT** prepend the entry, just append it, you do not want the config to be wrong and you lose the connection to the box entirely. Aliyun makes it extra tedious to re-install the box to its official images.

## Reboot and try
Open a new tab from Aliyun's ECS console to establish a VNC session. Keep it open and open another SSH connection to the running ECS.

On your SSH session, reboot the ECS:
```
sudo reboot
```
On your VNC session, hold downarrow (or any other key that would interrupt GRUB's autoboot) to interrupt GRUB

Choose entry `ArchISO (x86_64)` and enter, this should boot into the ISO.

If your boot is successful and your config is correct, you should have the following `lsblk` output on the Arch console:
```
> lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop1         7:1    0 685.2M  1 loop /run/archiso/airootfs
vda         254:0    0    40G  0 disk 
├─vda1      254:1    0     1M  0 part
├─vda2      254:2    0   191M  0 part
└─vda3      254:3    0  39.8M  0 part
```
And check `mount` to make sure there's nothing mounted from the main  disk. This is important as any mounting from the disk would hold you from re-partitioning it.

Also check `ip a`, `ip r` to confirm the you have successfully got IP address and route.

If there's anything wrong, you can go back to your old distro by `reboot` and fix them up. 

## (Optional) Connect with SSH
Generate the pubkey from your local private key
```
ssh-keygen -y -f .ssh/keys/by-host-user/ali-root.pem
```
Update the remote `.ssh/authorized_keys` to contain only your pubkey. Ali's VNC function "Copy command input" would be of great help here.

You can then connect with SSH, but you'll need to drop a `known_host` entry first if you've connected to the ECS before.

## Repartition the disk
You can partition the disk in any way you want it, following [the Installation Guide](https://wiki.archlinux.org/title/Installation_guide#Partition_the_disks)

But for maximum space utilization, I would recommend to create a single-partition MSDOS table, and use that single partition as the fs you would store root.

```
echo 'label: dos
start=2048' | sfdisk /dev/vda
```

Offset `2048` sectors is the default `fdisk` would use, and leaving a 1MiB speace before the first partition should be enough for GRUB to use.

## Format the partition
You can format the partition following [the Installation Guide](https://wiki.archlinux.org/title/Installation_guide#Format_the_partitions)

But for agile mounting and robust snapshot support, I would recommend to create a `Btrfs` on it:
```
mkfs.btrfs /dev/vda1
```

Before we actually mount it for installation, we would first create some subvolumes, what to create and how to setup them is up to yourself, but here's my setup:
```
mount /dev/vda1 /mnt
cd /mnt
for subvol in root home var swap; do
    btrfs subvolume create @${subvol}
done
chattr +C @{var,swap}
cd @swap
btrfs filesystem mkswapfile -s 4G swapfile
cd /
umount /mnt
```

## Mount the filesystem(s)
Still, you can follow the [Installation Guide](https://wiki.archlinux.org/title/Installation_guide#Mount_the_file_systems)

But for my setup, that would need to be mounted like this:
```
mount /dev/vda1 /mnt -o subvol=@root
for subvol in home var swap; do
    mount /dev/vda1 /mnt/${subvol} -o subvol=@${subvol} --mkdir
done
swapon /mnt/swap/swapfile
```

## Installation
This shall be done pretty much the same as the [Installation Guide](https://wiki.archlinux.org/title/Installation_guide#Installation), but note the following differences:
- As we're installing for a server that directly faces the Internet (albeit behind Aliyun's 1:1 NAT), it's recommended to:
  - Choose a safe variant to every package. Namely I'd recommend to choose `linux-hardended` over `linux` or `linux-lts`
  - Install a firewall and enable it unless you know what you're doing. I'd recommend `nftables`, which comes with a pretty sane default config.
  - Pre-configure your services with least trust to the Internet. Open only the necessary parts.
- Unless you don't trust Aliyun (and in that case you shouldn't even rent their ECS), you should use their mirror which is not neccessarily faster but would save the bandwidth usage of other mirror sites. Edit `/etc/pacman.d/mirrorlist` and prepend the following server:
    ```
    Server = http://mirrors.cloud.aliyuncs.com/archlinux/$repo/os/$arch
    ```


The following is my `pacstrap` command:
```
pacstrap /mnt base linux-hardened grub btrfs-progs vim sudo openssh nftables
```

I intall these packages with their corresponding reasons:
- `base` is the necessary meta package which pulls in other essential packages to ensure a usable system
- `linux-hardened` is the safe kernel, it has less vulnerabilities.
- `grub` is the bootloader, and it also supports reading from `Btrfs` natively. `syslinux` is another possible choice for legacy BIOS booting, but it has more limitations.
- `btrfs-progs` provides userspace `btrfs` tool, and it also provides `fsck.btrfs` so `mkinitcpio` would be happy (although it does nothing)
- `vim` is the text editor
- `sudo` is necessary to operate without root account, of which you would not really want to open access to Internet
- `openssh` is the SSH server
- `nftables` is the firewall, powerful yet sane

The following packages which are deemed necessary on a normal installtion are not installed:
- `linux-firmware` as it is not needed for virtual hardware
- `intel-ucode` or `amd-ucode` as a VM cannot upload the ucode patch to physical CPU (or could but that would not be of good use, as everything is already too late)

Enter the chroot and continue the chroot installtion as your normally do, following [the Guide](https://wiki.archlinux.org/title/Installation_guide#Installation).
```
arch-chroot /mnt
```

The following configs are what I would do differently or not includeded in the guide:
- Use a UTC timezone instead of your local time. As I manage a lot of servers across the world I don't want to flip between different timezones constantly, that's too much headache.
    ```
    ln -sf /usr/share/zoneinfo/UTC /etc/localtime
    ```
- No need to sync to the RTC, it's kept in sync by the hypervisor.
- Enable only the `en_US.UTF-8 UTF-8` locale, and set that as lang. For non-desktop usage other locales are not of good use.
- No need to re-generate the initramfs
- No need to setup passwd for root, unless you're sure you would need to login as `root`.
- Create a non-root user and add a supplementary group `wheel` for it so it could use `sudo`, and remember to set a password for it
    ```
    useradd --create-home nomad7ji
    usermod --append --groups wheel nomad7ji
    passwd nomad7ji
    ```
- Remember to `visudo` to allow `wheel` group to use `sudo`
- Modify `/etc/ssh/sshd_config` in chroot to disable password authentication and root login, and set private key file to a place only root is able to write. And only allow your own user to login.
    ```
    PermitRootLogin no
    AuthorizedKeysFile      /etc/.ssh/authorized_keys/%u
    PasswordAuthentication no
    AllowUsers  nomad7ji
    ```
- Pre-populate your pubkey like the following
    ```
    echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCbWgrc2ekP5JNVDar5olAnvsAgsOSUCqwhpd5VSSNBpsH1oYiv/vhcQ33oPzZn/hzfCDzJpQELEkY9XBSTu0DPk6XEYXMzikf7i6QRM7APc3mqfBWjKPa/kLIWyGr2gPMczQYYe2DbaEexCjJaW+sbgcDGZUSsNlXs16ngn7VO5i0rCsq5EaE7wsdVjlwK3F0ABnBn1YQF70Jt8inWU+jRHZiNM5f4odumpvmp87LfYHJc+DDP21fUN5kbPirXQCmRv9Y4FJTZg+5mv1VUWzTKIpRNsbmBTyfqfG2PjKV7kaYEYnhbxM5nV/LrmLpIxRGQUqFNpJ9zkXPa5u7aZ7alG+gCbVOIzcC6VehRBKXOB3r62OwoaB8jmK2USbwjjwh/OyFYhMkHgeV0P/9X6D5Aa7hOygoP6hJKHhEU0DuHhNt4BMK+fyui34OojaHxUxStJDoB9T+3spISW0iHu3V8RLGFVPft9yWA8laz0zWEswj5faFfhhxQcpVAwwGS78s= nomad7ji@iZ2zeg7ccxt693g9mtu5ioZ' |
        install -DTm400 -o nomad7ji -g nomad7ji /dev/stdin /etc/.ssh/authorized_keys/nomad7ji
    ```
- Install GRUB to `vda` and generate the config for it:
    ```
    grub-install /dev/vda
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
- Create a systemd-networkd config file `/etc/systemd/network/20-vpc.network` for DHCPv4 that match on the virtual NIC:
    ```
    [Match]
    Name=ens*

    [Network]
    DHCP=ipv4
    ```
- Run the `resolv.conf` provided by `systemd-resolved`:
    ```
    ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
    ```
- Remember to enable the necessary systemd units:
    ```
    systemctl enable sshd systemd-{network,resolve}d nftables
    ```

## Finish
Exit from the chroot and umount everything under `/mnt`, and reboot to enjoy ArchLinux on Aliyun. You should be able to `ssh` directly into the box with your already set key, although you'll need to drop a `known_host` entry if you've connected to the ECS before.