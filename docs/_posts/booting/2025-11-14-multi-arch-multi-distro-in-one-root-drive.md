---
layout: post
title:  "Multi-architecture multi-distro in one root partition"
date:   2025-11-14 18:30:00 +0800
categories: booting
---

Recently I needed to do offline OS maintainance work on quite a few of my devices, for which I used Ventoy + archiso on x86_64 for Debian 13 / Arch, and ALARM os drive on aarch64 for Debain 13 / ALARM. For which I find archiso more and more annoying as I had to re-do a lot of initial setups.

While I know these could be improved if I have a dedicated persistent fs for configs, or cloud-init scripts, or archiso boot parameters, I don't quite want immutable live system for the work any more. So, I decided, what if I have a single drive, on which I have all of the following systems booting from the same root partition?

- Arch Linux x86_64 bootable via both UEFI and legacy
- Debian 13 x86_64 bootable via both UEFI and legacy
- Debian 13 aarch64 bootable via UEFI (and U-boot distroboot)
- Arch Linux ARM aarch64 bootable via both UEFI (and U-boot distroboot)
- And of course more!

## Background knowledge

Before the actual installation, I'll explain the background knowledge first. If you don't bother with, skip to [next chapter](#drive-preparation)

### Booting on x86_64 UEFI, without CSM

On x86_64 UEFI, without CSM, the booting process is quite simple:

1. UEFI powers on and do preparation until booting logic is ready
2. If quick boot is enabled, load only necessary drivers, and execute the first available (destination exists and binary exists) BootEntry in current BootOrder, if this succeeds then no remaining steps would be executed
3. If external boot sources are available (e.g. network adapters with UEFI ROM) and not disabled, let them scan and register BootEntrys as needed; on most devices this step does not execute at all
4. UEFI BIOS loads various drivers and scans for all drives for EFI partition, that is, MSDOS type "EFI (FAT-12/16/32)", or GPT type "C12A7328-F81F-11D2-BA4B-00A0C93EC93B", for each FAT fs with such type, open and search for EFI binary at removable path "EFI/BOOT/BOOTX64.EFI", and register a BootEntry with generated name for it like "UEFI OS"; for different UEFI vendors the logic whether to register them with order earlier or later than existing entries are undetermined
4. Go similarly as step 2

Note about MBR on UEFI: as the specification only required support for GPT, the support or no-support for MBR is undeterminable before you get your hands on the actual machine. Windows and systemd-boot simply refuses to install on MBR on UEFI. While all of my devices support such and I use it for local system installation on small drives, I would only focus on GPT on UEFI due to the fact that we want the result drive bootable on variuos machines.

Most of UEFI-compatible boot managers support to be (or has to be at least) installed at removable path. E.g. for grub (expecting EFI partiton mounted at `/efi`):

```
grub-install --target x86_64-efi --removable
```

On Debian 13 this installs the following files to `/efi/EFI/BOOT`:
- `BOOTX64.CSV`: this contains required entry to be registered once shim at `BOOTX64.EFI` successfully booted
- `BOOTX64.EFI`: Debian's signed shim, loads signed `grubx64.efi`, and register entries according to `BOOTX64.CSV`; the one UEFI firmware would pick as removable EFI
- `grub.cfg`: Grub's config, it just tells grub to scan for real root and look up configs there. An example content:
    ```
    search.fs_uuid 91c69930-b508-4a42-b510-d63544d7eae0 root hd1,gpt2
    set prefix=($root)'/boot/grub'
    configfile $prefix/grub.cfg
    ```
    Grub's config files look like shell scripts and you can imagine `configfile` as `source` in shell, so in this case the `grub.cfg` in EFI partition just records where to find the root partition (search for fs with uuid `91c69930-b508-4a42-b510-d63544d7eae0` and records the result in variable `root`, if failed then use default `hd1,gpt2`), sets another variable `prefix`, being the path to folder `/boot/grub` under that root fs, then "source" another config `grub.cfg` under there.
- `grubx64.efi`: Grub's core EFI binary, signed by Debian, loads `grub.cfg`
  - The file would carry a built-in `$prefix` variable equalling `/EFI/debian` to instruct where to look up for a grub folder; in removable case it would instead try `[ESP]/EFI/BOOT`, so it looks up `grub.cfg` here

- `mmx64.efi`: Machine owner key manager, only needed for secure boot, not needed for fully removable use cases, can be safely deleted

On Arch Linux this installs only `BOOTX64.EFI`, even the config needs to be manually created.

For a removable drive, on which we would have (some) kernels unsigned, we certainly would not want strict Secure Boot, neither would we want permissive Secure Boot with Machine Owner Key managed by ourselves. And we would not want it to register non-removable UEFI BootEntry s if possible.

The dependency tree in Debain's Grub split packages make it really hard to install secure-boot-less under UEFI, as grub-efi-amd64-bin, a soft depened making `grub-install --target x86_64-efi` possible, hard depends grub-efi-amd64-signed. And the whole dependency tree becomes locked-in and almost impossible to uninstall due to they considered "essential". And dpkg hook would "friendly" help use to re-install (update) Grub on version change, not respecting the existing layout. So in later steps we would install and manage Grub the boot loader part from Arch Linux, and install only the booting configuration generation part on Debian.

Don't worry about "Debian not having its own Grub". It is `$prefix/grub.cfg` that normally `grub-mkconfig` / `update-grub` updates, these are managed by sytem themselves, and the configuration tool would still be installed.

Other than the per-system `grub.cfg`, we would want an "outer grub.cfg", which is directly used by Grub, containing `menuentry` to instruct which sub `grub.cfg` to redirect to.

The reason we choose Grub rather than other booting methods:
- While systemd-boot is also a candidate on x86_64 UEFI, the straight-forward installtion tool `bootctl` does not natively support a removable option (so it must write a BootEntry **at installation** which I find quite annoying); although you could manually copy the binary to the removable EFI path, no maintainance can be done easily with `bootctl update`; also systemd-boot does not support legacy BIOS so we have to maintain more booting configs, which is a hassle itself
- Placing a unified kernel image to the removable place is OK if you only want a single distro, but is impossible if we want multiple distro (well technically you can use kernel image cross-distro, but good luck updating them)

In summary, for the multi-boot logic we would need: GPT + EFI partition + one single removable Grub EFI binary + one single `grub.cfg` as `menuentry` selector + one `grub.cfg` per system maintained by the system itself

### Booting on x86 legacy BIOS or on x86_64 UEFI with CSM

This is still "simple" but not straight-forward in the modern perspective. Still, let's write the main ideas down:

1. BIOS powers on and do preparation until booting logic is ready
2. BIOS registers newly found drives into its pool, not necessarily at last positions
3. For each target in the booting order configuration, if not drive, delegate to external source (e.g. network manager with booting ROM), otherwide load the Master Boot Record (MBR) on sector 0 and try to execute it; in most cases this wouldn't return even if the MBR is not technically bootable (some partition tools would place a binary here to print "unbootable device", and for some this means hang i.e. soft locked)
4. If all drives in booting order failed, print "no bootable drives found" and hang

Note while `MBR` was mentioned above, the partition table on the drive does not have to be `MBR / msdos`. The BIOS actually knows nothing above the partition but rather just reads stuff from fixed offsets (think that as "as-if MBR0", in fact the whole drive can be a "super floppy" i.e. fs on whole drive, and as long as sector 0 is available it does not matter).

So most of the legacy-BIOS-compatible boot loaders need to be installed to MBR, or also technically the whole drive, e.g. for Grub:

```
grub-install --target i386-pc /dev/sda
```

The binary that Grub installs into MBR / sector 0 is called `boot.img` by Grub itself, the functionality is similar to `grubx64.efi` in the UEFI case: to load necessary drivers to look up real `grub.cfg`. But as MBR sector 0 is too small (512 Byte) it's impossible to do it by the small `boot.img` itself. For this another part of binary called `core.img` needs to be looked up and executed. Grub does this differently depending on whether you're booting on MBR or GPT:

- on MBR, `core.img` is stored from sector 1 onward, before first partition, the space is 1 MiB - 512B, and in real world would never be fully utilized.
- on GPT, `core.img` is stored in the partition with type `BIOS boot` (21686148-6449-6E6F-744E-656564454649), for the same reason this can be only 1 MiB

Of course `core.img` itself would carry metadata including how large the actual data is, fs drivers so actual root partition could be opened, and optionally a built-in config.

The logic is simply, `boot.img` -> `core.img` -> `$prefix/grub.cfg`, after that latter steps are similar to UEFI cases.

Similar as UEFI, we would want a single outer Grub with config managed by ourselves to function as selector, and each system managing their own inner `grub.cfg` (but not having their Grub boot manager) ready to be picked up by ourselves.

The reason we choose Grub rather than other booting methods:
- While syslinux can also provide menu, and is pretty KISS, it would need seperate config, instead of reusing the same config as UEFI

In summary, for the multi-boot logic we would need: GPT + BIOS Boot partition + one single Grub binary in MBR0 and BIOS Boot partition + one single `grub.cfg` as `menuentry` selector + one `grub.cfg` per system maintained by the system itself

### Booting on AArch64, UEFI or U-Boot distroboot

Some AArch64 devices support UEFI, many others don't and use U-Boot. In not-too-old U-Boot builds, the "distroboot" concept scans for various bootable targets, and removable EFI binary `EFI/BOOT/BOOTAA64.EFI` is one of them, besides `extlinux.conf` and `boot.scr(.uimg)` . I build u-boot myself on all of SBCs and TV boxes on my hand and they all support such without explicitly enabling. As we want multi-boot on a single drive on both AArch64 and x86_64 we would only focus the UEFI and distroboot UEFI.

On AArch64 UEFI, the steps are similar as [x86_64](#booting-on-x86_64-uefi-without-csm), except that the fallback EFI binary name is `BOOTAA64.EFI`

On AArch64 U-Boot that chainloads removable EFI in distroboot logic:
1. U-Boot powers on and do preparation until booting logic is ready
2. Loads environment variables either from persistent storage or from built-in, in both cases to memory
3. Run environment variable `bootcmd`, in modern cases, `bootcmd='bootflow scan -lb'`
4. So, run `bootflow`, the argument `scan` means to scan all possible sources, in most cases these include all block devices first, then network; the argument `-l` means to print each scanned bootablt target; the argument `-b` means for each target scanned, try to boot immediately.
5. Let's only focus on removable/fallback EFI binary, and assumes it being the only possible target and scanned
6. Prepare some "UEFI" environments and "UEFI services", then loads the EFI binary in, then execute it.

Note specifically for per-device DTB to be applied correctly in the U-Boot case, if `/boot` is in root fs, the job cannot be done by Grub (cannot expect a partition to be readable before you could even tell there's a block device), rather the DTB has to be loaded by U-Boot

## Drive preparation

Boot archiso or do this in a device already running Linux.

Run your perferred partition tool to partition the drive with the following partitions:
- 100 MiB EFI system partition
- 1 MiB BIOT boot partition
- Remaining as a single root partition
- Others as you like

Or simply save the following infos in a temporary file e.g. `parts.info`:

```
label: gpt
unit: sectors
sector-size: 512

size=204800, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B
size=2048, type=21686148-6449-6E6F-744E-656564454649
type=0FC63DAF-8483-4772-8E79-3D69D8477DE4
```

Then run `sfdisk` to format use the info:

```
sfdisk /dev/[drive] < parts.info
```

The result partitions shall look like the following (`/dev/vda` is used in the following example):

```
Checking that no-one is using this disk right now ... OK

Disk /dev/[drive]: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Created a new GPT disklabel (GUID: D8997F52-AFD6-4A0F-845F-FF12CDD718F6).
/dev/[drive]1: Created a new partition 1 of type 'EFI System' and of size 100 MiB.
/dev/[drive]2: Created a new partition 2 of type 'BIOS boot' and of size 1 MiB.
/dev/[drive]3: Created a new partition 3 of type 'Linux filesystem' and of size 63.9 GiB.
/dev/[drive]4: Done.

New situation:
Disklabel type: gpt
Disk identifier: D8997F52-AFD6-4A0F-845F-FF12CDD718F6

Device      Start       End   Sectors  Size Type
/dev/[drive]1    2048    206847    204800  100M EFI System
/dev/[drive]2  206848    208895      2048    1M BIOS boot
/dev/[drive]3  208896 134215679 134006784 63.9G Linux filesystem

The partition table has been altered.
```

Create a FAT fs on the ESP:
```
mkfs.vfat /dev/[drive]1
```

Create a Btrfs on the root partition:
```
mkfs.btrfs /dev/[drive]3
```

Note about Btrfs compression: if you want it, you could set it at fs-creation time, e.g.:
```
mkfs.btrfs --compress zstd:15 /dev/[drive]3
```
But the later steps assume this not set and we would write the compression level manually at mount time and in fstab. You can omit those if you did this at fs-creation time.

Now mount the Btrfs root partition to somewhere, we need to create some subvols
```
mount --mkdir /dev/[drive]3 /mnt/manyos
cd /mnt/manyos
```

Let's create subvols. The main focus is that we want seperate root volumes for each system, and shared home. (`@` or `+` is not strictly needed in subvol names, but it helps to tell them from plain folders)
- The generic one subvol per system except shared home style (very simple for latter mounting and fstab):
  ```
  btrfs subvolume create @arch @debian @home
  ```
- My style:
  ```
  mkdir shared arch-x86_64 debian-x86_64 alarm-aarch64 debian-aarch64
  btrfs subvolume create shared/@{home{,_.snapshots},etc_ssh} {arch-x86_64,debian-x86_64,alarm-aarch64,debian-aarch64}/{@{,.snapshots},+nocow}
  chattr +C {arch-x86_64,debian-x86_64,alarm-aarch64,debian-aarch64}/+nocow
  mkdir -p arch-x86_64/+nocow/var/{cache,log,spool,tmp}
  chmod 1777 arch-x86_64/+nocow/var/tmp
  mkdir template
  tar -f template/nocow.tar -C arch-x86_64/+nocow -cv .
  for i in arch-x86_64/@ {debian-x86_64,alarm-aarch64,debian-aarch64}/{@,+nocow}; do tar -f template/nocow.tar -C $i -xv; done
  ```
  The layout shall look the the following:
  ```
  > tree
  .
  ├── arch-x86_64
  │   ├── @
  │   │   └── var
  │   │       ├── cache
  │   │       ├── log
  │   │       ├── spool
  │   │       └── tmp
  │   ├── +nocow
  │   │   └── var
  │   │       ├── cache
  │   │       ├── log
  │   │       ├── spool
  │   │       └── tmp
  │   └── @.snapshots
  ├── ...
  ├── shared
  │   ├── @home
  │   └── @home_.snapshots
  └── template
      └── nocow.tar

  34 directories, 1 file
  ```
  The benefit of the above layout is that stuffs not needing snapshot and compression are enclosed into a single `+nocow` subvol and bind-mounting is used instead of many subvols; and system-specific stuffs are enclosed into one single top-level folder, and shared stuffs not so, and .snapshots subvol for snapper is pre-created.

In later steps I would follow only my own style

## Installing Arch Linux x86_64

As discussed earlier we would use Arch Linux as the one to install and maintain Grub x86_64 (as, Grub almost HAS TO BE installed as Secure Boot with shim, which we don't need at all), so let's install Arch Linux first.

Mainly you shall follow [the official Installation Guide](https://wiki.archlinux.org/title/Installation_guide) for the most part, to boot archiso on x86_64 and install in UEFI style; of course you could also do this on a device already running Linux. I would only cover the archiso case.

Let's focus on things that shall go differently from the official way:

1. Follow the official guide, until before "Partition the disks"
2. Skip "Partition the disks"
3. Skip "Format the partitions"
4. To mount my layout, mount root and nocow subvol first:
    ```
    mount -o compress=zstd:15,subvol=arch-x86_64/@ --mkdir /dev/[drive]3 /mnt/root
    mount -o subvol=arch-x86_64/+nocow --mkdir /dev/[drive]3 /mnt/arch-x86_64+nocow
    ```
    Then pre-create fstab and edit it:
    ```
    mkdir /mnt/root/etc
    cp /etc/fstab /mnt/root/etc/
    vim /mnt/root/etc/fstab
    ```
    Remember to use vim's functionality to do multi-line edit (shift + v, crtl + v, etc) and the ability to pipe content into external command (select in multi-line visual, then `:`, then `!column -t` to force a table look)

    The content shall look like the following (remember to use `blkid` to acquire your real UUID for root partition and ESP)
    ```
    # Static information about the filesystems.
    # See fstab(5) for details.

    # <file system> <dir> <type> <options> <dump> <pass>
    UUID=6894094c-a75e-4f1a-b228-283faf7bf003  /                       btrfs  rw,compress=zstd:15,subvol=arch-x86_64/@            0  0
    UUID=6894094c-a75e-4f1a-b228-283faf7bf003  /.snapshots             btrfs  rw,compress=zstd:15,subvol=arch-x86_64/@.snapshots  0  0
    UUID=6894094c-a75e-4f1a-b228-283faf7bf003  /home                   btrfs  rw,compress=zstd:15,subvol=shared/@home             0  0
    UUID=6894094c-a75e-4f1a-b228-283faf7bf003  /home/.snapshots        btrfs  rw,compress=zstd:15,subvol=shared/@home_.snapshots  0  0
    UUID=6894094c-a75e-4f1a-b228-283faf7bf003  /mnt/arch-x86_64+nocow  btrfs  rw,compress=zstd:15,subvol=arch-x86_64/+nocow       0  0
    /mnt/arch-x86_64+nocow/var/cache           /var/cache              none   bind,private                                        0  0
    /mnt/arch-x86_64+nocow/var/log             /var/log                none   bind,private                                        0  0
    /mnt/arch-x86_64+nocow/var/spool           /var/spool              none   bind,private                                        0  0
    /mnt/arch-x86_64+nocow/var/tmp             /var/tmp                none   bind,private                                        0  0
    UUID=5D03-ED7A                             /efi                    vfat   rw,noatime                                          0  2
    ```

    Then mount everything remaining up:
    ```
    mount --all --fstab /mnt/root/etc/fstab --target-prefix /mnt/root --mkdir
    ```

    A `lsblk` shall look like following now:
    ```
    # lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    vda    254:0    0   128G  0 disk
    ├─vda1 254:1    0   100M  0 part /mnt/root/efi
    ├─vda2 254:2    0     1M  0 part
    └─vda3 254:3    0 127.9G  0 part /mnt/root/var/tmp
                                     /mnt/root/var/spool
                                     /mnt/root/var/log
                                     /mnt/root/var/cache
                                     /mnt/root/mnt/arch-x86_64+nocow
                                     /mnt/root/home/.snapshots
                                     /mnt/root/home
                                     /mnt/root/.snapshots
                                     /mnt/arch-x86_64+nocow
                                     /mnt/root
    ```
5. Continue from "Installation", I'd recommend to choose the following bootstrap packages (pre-configure booster so initramfs is only generated once for universal, skipping the non-universal one):
    ```
    echo 'universal: true' > /mnt/root/etc/booster.yaml
    pacstrap -K /mnt/root base booster linux intel-ucode amd-ucode linux-firmware btrfs-progs dosfstools grub vim sudo
    ```
6. Skip "Configure the system / Fstab", the fstab generated on our Btrfs layout is pretty messy, just use our own
7. Continue from "Chroot", `arch-chroot /mnt/root` and do the remaining parts, until before "Network configuration"
8. For "Network configuration", the hostname could be unique for each system, or same, depending on your need; for network manager, I'd recommend just use `systemd-networkd`, so:
    - Enable networkd and resolved: `systemctl enable systemd-{network,resolve}d`
    - Quit from chroot
    - Copy archiso's network config files: `cp -rva /etc/systemd/network/* /mnt/root/etc/systemd/network/`
    - Re-link resolv.conf `ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/root/etc/resolv.conf`
    - Re-enter chroot
9. Skip "Initramfs", we're using `booster` instead of the default `mkinitcpio`, and the universal initramfs was already created
10. For "Boot Loader", we would install grub, the package was already installed into root in earlier bootstrap steps, we only need to install it to bootable:
    1. Install as removable EFI, note we also specify `--boot-directory /efi`, so `grub` modules and first-stage config are saved and loaded from there. We would only want each system's `/boot` to store there boot config
        ```
        grub-install --removable --efi-directory /efi --boot-directory /efi
        ```
    2. Install to MBR, similarly note we also specify `--boot-directory /efi`
        ```
        grub-install --target i386-pc --boot-directory /efi /dev/[drive]
        ```
    3. Hack/fix `/etc/grub.d/10_linux` so it would prefer booster initramfs (without this, booster initramfs would be hidden in a submenu) and ro root; if you're not using booster only or you don't require ro root on boot, you can skip this:
        ```
        From 9ad850b2b8842bb673313be08f6a5af66cdf12ea Mon Sep 17 00:00:00 2001
        From: Guoxin Pu <pugokushin@gmail.com>
        Date: Thu, 13 Nov 2025 15:52:27 +0800
        Subject: [PATCH] use booster as main initramfs and prefer ro

        ---
        10_linux | 26 ++------------------------
        1 file changed, 2 insertions(+), 24 deletions(-)

        diff --git a/10_linux b/10_linux
        index e16cea8..3ce3a9d 100755
        --- a/10_linux
        +++ b/10_linux
        @@ -147,7 +147,7 @@ linux_entry ()
          message="$(gettext_printf "Loading Linux %s ..." ${version})"
          sed "s/^/$submenu_indentation/" << EOF
                echo    '$(echo "$message" | grub_quote)'
        -       linux   ${rel_dirname}/${basename} root=${linux_root_device_thisversion} rw ${args}
        +       linux   ${rel_dirname}/${basename} root=${linux_root_device_thisversion} ro ${args}
        EOF
          if test -n "${initrd}" ; then
            # TRANSLATORS: ramdisk isn't identifier. Should be translated.
        @@ -227,7 +227,7 @@ for linux in ${reverse_sorted_list}; do
          done

          initrd_real=
        -  for i in "initrd.img-${version}" "initrd-${version}.img" \
        +  for i in "booster-${version}.img" "initrd.img-${version}" "initrd-${version}.img" \
                  "initrd-${alt_version}.img.old" "initrd-${version}.gz" \
                  "initrd-${alt_version}.gz.old" "initrd-${version}" \
                  "initramfs-${version}.img" "initramfs-${alt_version}.img.old" \
        @@ -304,28 +304,6 @@ for linux in ${reverse_sorted_list}; do
          linux_entry "${OS}" "${version}" advanced \
                      "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"

        -  if test -e "${dirname}/initramfs-${version}-fallback.img" ; then
        -    initrd="${initrd_early} initramfs-${version}-fallback.img"
        -
        -    if test -n "${initrd}" ; then
        -      gettext_printf "Found fallback initrd image(s) in %s:%s\n" "${dirname}" "${initrd_extra} ${initrd}" >&2
        -    fi
        -
        -    linux_entry "${OS}" "${version}" fallback \
        -                "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"
        -  fi
        -
        -  if test -e "${dirname}/booster-${version}.img" ; then
        -    initrd="${initrd_early} booster-${version}.img"
        -
        -    if test -n "${initrd}" ; then
        -      gettext_printf "Found booster initrd image(s) in %s:%s\n" "${dirname}" "${initrd_extra} ${initrd}" >&2
        -    fi
        -
        -    linux_entry "${OS}" "${version}" booster \
        -                "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"
        -  fi
        -
          if [ "x${GRUB_DISABLE_RECOVERY}" != "xtrue" ]; then
            linux_entry "${OS}" "${version}" recovery \
                        "${GRUB_CMDLINE_LINUX_RECOVERY} ${GRUB_CMDLINE_LINUX}"
        --
        2.51.2
        ```
    4. Generate `grub.cfg` for Arch Linux as how you do it on a normal installation:
        ```
        mkdir /boot/grub
        grub-mkconfig -o /boot/grub/grub.cfg
        ```
        Note this would not be the first config Grub loads, but rather just a "to-be-included" config.
    5. Let's write a real, outer config for Grub to achieve menu logic by `vim /efi/grub/grub.cfg` with content like following:
        ```
        search.fs_uuid 6894094c-a75e-4f1a-b228-283faf7bf003 root hd0,gpt2
        terminal_input console
        terminal_output console
        set suffix='@/boot/grub/grub.cfg'
        menuentry 'Arch Linux (x86_64)' {
                configfile ($root)/arch-x86_64/$suffix
        }
        ```
  11. Finalize the installation, umount everything and poweroff

After the above steps we shall have a UEFI + legacy bootable, you can boot on different machines to validate it, the boot menu shall look like this on both UEFI and legacy:

```

                         GNU GRUB  version 2:2.14rc1-2

 ┌────────────────────────────────────────────────────────────────────────────┐
 │*Arch Linux (x86_64)                                                        │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘

      Use the ▲ and ▼ keys to select which entry is highlighted.          
      Press enter to boot the selected OS, `e' to edit the commands       
      before booting or `c' for a command-line.                           
                                                                             
```

And after pressing Enter it's the same old Arch Linux Grub menu as always.

While we're at it, you can of course add timeout, decoration, etc to the outer menu. I'll stick with the simple look and continue.

## After-installation for Arch Linux x86_64

This part is mostly about quality of life improvement and can be skipped. If you want to follow, boot into the installed Arch Linux.

### sshd config + key

```
pacman -S openssh
systemctl enable --now sshd
```

Verify that sshd pubkeys are generated successfully:

```
# ls /etc/ssh
moduli	ssh_config  ssh_config.d  sshd_config  sshd_config.d  ssh_host_ecdsa_key  ssh_host_ecdsa_key.pub  ssh_host_ed25519_key	ssh_host_ed25519_key.pub  ssh_host_rsa_key  ssh_host_rsa_key.pub
```

```
# systemctl status sshdgenkeys
○ sshdgenkeys.service - SSH Key Generation
     Loaded: loaded (/usr/lib/systemd/system/sshdgenkeys.service; disabled; preset: disabled)
     Active: inactive (dead) since Thu 2025-11-13 16:26:26 CST; 1min 51s ago
 Invocation: ef2323bd8d4e4007be02a0e3208f3c31
    Process: 578 ExecStart=/usr/bin/ssh-keygen -A (code=exited, status=0/SUCCESS)
   Main PID: 578 (code=exited, status=0/SUCCESS)
   Mem peak: 1.7M
        CPU: 84ms

Nov 13 16:26:25 dud systemd[1]: Starting SSH Key Generation...
Nov 13 16:26:26 dud ssh-keygen[578]: ssh-keygen: generating new host keys: RSA ECDSA ED25519
Nov 13 16:26:26 dud systemd[1]: sshdgenkeys.service: Deactivated successfully.
Nov 13 16:26:26 dud systemd[1]: Finished SSH Key Generation.
```

Do required sshd_config modification as needed:
```
vim /etc/ssh/sshd_config
```

- I would replace `AuthorizedKeysFile .ssh/authorized_keys` -> `AuthorizedKeysFile  /etc/ssh/authorized_keys/%u`, so no one can add their SSH pubkey except root (me); and I would prepare this folder as needed
- I would uncomment `#PasswordAuthentication` and set `PasswordAuthentication no` so password login is disabled

After the modification restart sshd so the changes take effect:

```
systemctl restart sshd
```

### snapshot on boot

For the setup I would want after-boot snapshots to be taken, so install `snapper`:

```
pacman -S snapper
```

Create configs and enable needed services
```
vim /etc/snapper/configs/root
vim /etc/snapper/configs/home
```

The content shall look like the following:
```
SUBVOLUME="/"
FSTYPE="btrfs"
QGROUP=""
SPACE_LIMIT="0.5"
FREE_LIMIT="0.2"
ALLOW_USERS=""
ALLOW_GROUPS=""
SYNC_ACL="no"
BACKGROUND_COMPARISON="yes"
NUMBER_CLEANUP="yes"
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="50"
NUMBER_LIMIT_IMPORTANT="10"
EMPTY_PRE_POST_CLEANUP="yes"
EMPTY_PRE_POST_MIN_AGE="1800"
```
```
SUBVOLUME="/home"
FSTYPE="btrfs"
QGROUP=""
SPACE_LIMIT="0.5"
FREE_LIMIT="0.2"
ALLOW_USERS=""
ALLOW_GROUPS=""
SYNC_ACL="no"
BACKGROUND_COMPARISON="yes"
NUMBER_CLEANUP="yes"
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="50"
NUMBER_LIMIT_IMPORTANT="10"
EMPTY_PRE_POST_CLEANUP="yes"
EMPTY_PRE_POST_MIN_AGE="1800"
```

And edit the global config to enable these profiles:
```
vim /etc/conf.d/snapper
```

With the following line:
```
SNAPPER_CONFIGS="root home"
```

By default `snapper-boot` only snapshots root, so modify the unit:
```
systemctl edit snapper-boot
```

The result shall look like this (ExecStart is appended after the original ExecStarted):
```
> cat /etc/systemd/system/snapper-boot.service.d/override.conf
[Service]
ExecStart=/usr/bin/snapper --config home create --cleanup-algorithm number --description "boot"
```

Then let's enable needed units:
```
systemctl enable --now snapper-{boot,cleanup}.timer
```

## Installing Debian x86_64

Let's do this on Arch Linux x86_64 with `debootstrap` and `arch-chroot`

Of course the tools shall be installed first:

```
pacman -S debootstrap arch-install-scripts
```

The installation goes similarly as Arch, but keep the following points in mind:
1. After chroot, do `export PATH=` with missing `/sbin` parts, whole command in later steps
2. Do not install any boot manager! The only system here that has a boot manager installed is Arch x86_64.

The steps are as follows:

1. Similarly, mount root and nocow subvol first:
    ```
    mount -o subvol=debian-x86_64/@ --mkdir /dev/[drive]3 /mnt/root
    mount -o subvol=debian-x86_64/+nocow --mkdir /dev/[drive]3 /mnt/debian-x86_64+nocow
    ```
    Then duplicate fstab:
    ```
    mkdir /mnt/root/etc
    sed 's/arch/debian/g' /etc/fstab > /mnt/root/etc/fstab
    ```
    Then mount everything remaining up:
    ```
    mount --all --fstab /mnt/root/etc/fstab --target-prefix /mnt/root --mkdir
    ```

    A `lsblk` shall look like following now:
    ```
    # lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    vda    254:0    0   128G  0 disk
    ├─vda1 254:1    0   100M  0 part /mnt/root/efi
    │                                /efi
    ├─vda2 254:2    0     1M  0 part
    └─vda3 254:3    0 127.9G  0 part /mnt/root/var/tmp
                                     /mnt/root/var/spool
                                     /mnt/root/var/log
                                     /mnt/root/var/cache
                                     /mnt/root/mnt/debian-x86_64+nocow
                                     /mnt/root/home/.snapshots
                                     /mnt/root/home
                                     /mnt/root/.snapshots
                                     /mnt/debian-x86_64+nocow
                                     /mnt/root
                                     /var/tmp
                                     /var/spool
                                     /var/log
                                     /home/.snapshots
                                     /var/cache
                                     /mnt/arch-x86_64+nocow
                                     /home
                                     /.snapshots
                                     /
    ```
2. Do `debootstrap` into the root
    ```
    debootstrap trixie /mnt/root http://[mirror_link]
    ```
3. chroot into `/mnt/root`, and set `PATH`
    ```
    arch-chroot /mnt/root
    ```
    ```
    export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    ```
    The PATH needs to be set as `arch-chroot` keeps PATH from Arch, which lacks `sbin`, but on Debian `sbin` is a seperate folder than `bin`
4. Install a few missing packages
    ```
    apt update
    apt install vim locales systemd-timesyncd btrfs-progs dosfstools linux-image-amd64 amd64-microcode intel-microcode firmware-linux
    ```
5. Do timezone, locale, hostname setup just like how you did it in Arch
6. For network manager, similarly, I'd recommend `systemd-networkd` + `systemd-resolved` pair, but on Debian `resolved` needs to be installed seperately:
    ```
    apt install systemd-resolved
    ```
7. Exit from chroot and borrow host Network configuration and re-link resolv.conf just like how we did for Arch x86_64 above, then re-enter chroot
8. Now, the Grub, we only need Debian to generate `grub.cfg`, but definitely not intalling and maintaining the actual Grub boot manager, so install only the system integration part and prepare the folder manually:
    ```
    apt install grub2-common
    mkdir /boot/grub
    vim /etc/default/grub
    ```
    With this setup there would be no pre-configured `grub` config, so use the following as a starting point:
    ```
    GRUB_DEFAULT=0
    GRUB_TIMEOUT=1
    GRUB_DISTRIBUTOR=`( . /etc/os-release && echo ${NAME} )`
    GRUB_CMDLINE_LINUX_DEFAULT="audit=0"
    GRUB_CMDLINE_LINUX=""
    GRUB_TERMINAL=console
    ```
    Remember to re-generate the one included by our outer grub
    ```
    update-grub
    ```
9. Exit from chroot and update our outer Grub config `/efi/grub/grub.cfg` to include a new menuentry:
    ```
    menuentry 'Debian (x86_64)' {
        configfile ($root)/debian-x86_64/$suffix
    }
    ```
10. Finalize the installation, umount everything and poweroff

After the above steps we shall have a UEFI + legacy bootable Arch Linux + Debian installation, you can boot on different machines to validate it, the boot menu shall look like this on both UEFI and legacy:

```

                         GNU GRUB  version 2:2.14rc1-2

 ┌────────────────────────────────────────────────────────────────────────────┐
 │*Arch Linux (x86_64)                                                        │
 │ Debian (x86_64)                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘

      Use the ▲ and ▼ keys to select which entry is highlighted.          
      Press enter to boot the selected OS, `e' to edit the commands       
      before booting or `c' for a command-line.                           
                                                                             
```

And after selecting Debian and pressing Enter it's the same old Debian Grub menu as always.

## After-installation for Debian x86_64

This part is mostly about quality of life improvement and can be skipped. If you want to follow, boot into the installed Debian and do similar things as in [After-installation for Arch Linux x86_64](#after-installation-for-arch-linux-x86_64), but with the following differences:

1. If you want to share hostname, it's better to share host keys, i.e. `/etc/ssh/ssh_host_*_key{,.pub}`, so clients would not complain about host key differing
2. For snapper, Debian comes with all units pre-enabled, disable the timeline as we want only on-boot: `systemctl disable --now snapper-timeline.timer`; and note the snapper config folder is the same but the global config is at `/etc/default/snapper`
3. While `debootstrap` still prepares an old-style APT `sources.list`, I recommend to migrate to new APT config style:
    ```
    rm /etc/apt/sources.list
    vim /etc/apt/sources.list.d/debian.sources
    ```
    With content like the following:
    ```
    Types: deb
    URIs: http://[mirror]/debian/
    Suites: trixie trixie-updates
    Components: main contrib non-free non-free-firmware
    Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

    Types: deb
    URIs: http://[mirror]/debian-security/
    Suites: trixie-security
    Components: main contrib non-free non-free-firmware
    Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

    Types: deb
    URIs: http://[mirror]/debian/
    Suites: trixie-backports
    Components: main contrib non-free non-free-firmware
    Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
    ```
4. It's good if you remember that we did not configure initramfs-tools to create generic initramfs. In most cases it comes with pre-configured `MODULES=most` and you would not need to modify that, only with `debian-install` could this be set to `MODULES=deps`. To verify, `vim /etc/initramfs-tools/initramfs.conf` and check `MODULES`. If it's `MODULES=deps` then modify it, save, and run `update-initramfs -u`
5. There're many firmware not installed by `firmware-linux` meta package, in most cases these are not needed, if you really really need all of them installed:
    ```
    apt install $(for i in $(apt-cache search '^firmware-' | cut -d ' ' -f 1); do dpkg-query -W -f='${Status}' $i &>/dev/null || echo $i; done | grep -v installer)
    ```

## Installing Arch Linux ARM aarch64

Let's do this on Arch Linux x86_64

1. Install dependencies first
    ```
    pacman -S qemu-user-static-binfmt arch-install-scripts
    ```
2. Duplicate host pacman config and do necessary modification
    ```
    cp /etc/pacman.conf pacman-alarm.conf
    vim pacman-alarm.conf
    ```
    1. Set `Architecture = aarch64` instead of `auto`
    2. Set repo `Inlucde = mirrorlist-alarm` instead of `/etc/pacman.d/mirrorlist`
    3. Temporarily set `SigLevel = Never` as we don't have ALARM keyring on Arch and adding those to Arch host would mess up the host, pacakges can later be re-verified once we have installed ALARM
    4. Add [my repo](https://github.com/7Ji/archrepo), we would use `linux-aarch64-7ji` as kernel, instead of ALARM's official `linux-aarch64`, the latter misses some drivers built-in and would not boot on some of my SBCs and it has max CPUs set to a small number so would not work nicely on VM either, and the worst is it has some naive hooks to always expect `mkinitcpio` instead of any other initramfs maker.
        ```
        [7Ji]
        Include = mirrorlist-3rdparty
        ```

3. Create mirrorlist for alarm
    ```
    vim mirrorlist-alarm
    ```
    With server:
    ```
    Server = http://[mirror]/archlinuxarm/$arch/$repo
    ```
4. Create mirrorlist for 3rdparty repo
    ```
    vim mirrorlist-3rdparty
    ```
    With server:
    ```
    Server = http://[mirror]/$repo/$arch
    ```
5. Mount the root tree
    ```
    mount -o subvol=alarm-aarch64/@ --mkdir /dev/[drive]3 /mnt/root
    mount -o subvol=alarm-aarch64/+nocow --mkdir /dev/[drive]3 /mnt/alarm-aarch64+nocow
    ```
    Then duplicate fstab:
    ```
    mkdir /mnt/root/etc
    sed 's/arch-x86_64/alarm-aarch64/g' /etc/fstab > /mnt/root/etc/fstab
    ```
    Then mount everything remaining up:
    ```
    mount --all --fstab /mnt/root/etc/fstab --target-prefix /mnt/root --mkdir
    ```

    A `lsblk` shall look like following now:
    ```
    # lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    vda    254:0    0   128G  0 disk
    ├─vda1 254:1    0   100M  0 part /mnt/root/efi
    │                                /efi
    ├─vda2 254:2    0     1M  0 part
    └─vda3 254:3    0 127.9G  0 part /mnt/root/var/tmp
                                     /mnt/root/var/spool
                                     /mnt/root/var/log
                                     /mnt/root/var/cache
                                     /mnt/root/mnt/alarm-aarch64+nocow
                                     /mnt/root/home/.snapshots
                                     /mnt/root/home
                                     /mnt/root/.snapshots
                                     /mnt/alarm-aarch64+nocow
                                     /mnt/root
                                     /var/tmp
                                     /var/spool
                                     /var/log
                                     /home/.snapshots
                                     /var/cache
                                     /mnt/arch-x86_64+nocow
                                     /home
                                     /.snapshots
                                     /
    ```
6. Do `pacstrap` into the root
    ```
    echo 'universal: true' > /mnt/root/etc/booster.yaml
    pacstrap -C pacman-alarm.conf -K -M /mnt/root base booster linux-aarch64-7ji linux-firmware btrfs-progs dosfstools grub vim sudo archlinuxarm-keyring 7ji-keyring
    ```
7. Re-edit (or duplicate from CWD) the pacman configs and mirrorlist under `/mnt/root/etc`, as now they come the official package, remember to add 7Ji repo
    ```
    vim /mnt/root/etc/pacman.conf
    vim /mnt/root/etc/pacman.d/mirrorlist
    vim /mnt/root/etc/pacman.d/mirrorlist-3rdparty
    ```
7. Chroot into the target and confirm we're running as aarch64
    ```
    arch-chroot /mnt/root
    ```
    ```
    uname -m
    ```
    Note: as we're using qemu-static, the stdin/out is technically not directly attached to our terminal, so text editing could be a pain. If text editing is needed, do it from another SSH session from host under `/mnt/root` instead so VIM and Nano could correctly write to terminal.
8. Initalize the keyring
    ```
    pacman-key --init
    pacman-key --populate
    ```
10. Let's actually verify the packages, now that we've nitialized the keyring
    ```
    pacman -Sy --downloadonly $(sed -n '/^%NAME%/{n;p}' /var/lib/pacman/local/*/desc)
    ```
11. Continue and finish the setup just like how we did in Arch Linux, until before the boot manager
12. Similar to Arch Linux x86_64, let's install and hack Grub
    1. Install as removable EFI, note we also specify `--boot-directory /efi`, so `grub` modules and first-stage config are saved and loaded from there. We would only want each system's `/boot` to store there boot config
        ```
        grub-install --removable --efi-directory /efi --boot-directory /efi
        ```
    2. Hack/fix `/etc/grub.d/10_linux` so it would prefer booster initramfs (without this, booster initramfs would be hidden in a submenu) and ro root; if you're not using booster only or you don't require ro root on boot, you can skip this:
        ```
        From 9ad850b2b8842bb673313be08f6a5af66cdf12ea Mon Sep 17 00:00:00 2001
        From: Guoxin Pu <pugokushin@gmail.com>
        Date: Thu, 13 Nov 2025 15:52:27 +0800
        Subject: [PATCH] use booster as main initramfs and prefer ro

        ---
        10_linux | 26 ++------------------------
        1 file changed, 2 insertions(+), 24 deletions(-)

        diff --git a/10_linux b/10_linux
        index e16cea8..3ce3a9d 100755
        --- a/10_linux
        +++ b/10_linux
        @@ -147,7 +147,7 @@ linux_entry ()
          message="$(gettext_printf "Loading Linux %s ..." ${version})"
          sed "s/^/$submenu_indentation/" << EOF
                echo    '$(echo "$message" | grub_quote)'
        -       linux   ${rel_dirname}/${basename} root=${linux_root_device_thisversion} rw ${args}
        +       linux   ${rel_dirname}/${basename} root=${linux_root_device_thisversion} ro ${args}
        EOF
          if test -n "${initrd}" ; then
            # TRANSLATORS: ramdisk isn't identifier. Should be translated.
        @@ -227,7 +227,7 @@ for linux in ${reverse_sorted_list}; do
          done

          initrd_real=
        -  for i in "initrd.img-${version}" "initrd-${version}.img" \
        +  for i in "booster-${version}.img" "initrd.img-${version}" "initrd-${version}.img" \
                  "initrd-${alt_version}.img.old" "initrd-${version}.gz" \
                  "initrd-${alt_version}.gz.old" "initrd-${version}" \
                  "initramfs-${version}.img" "initramfs-${alt_version}.img.old" \
        @@ -304,28 +304,6 @@ for linux in ${reverse_sorted_list}; do
          linux_entry "${OS}" "${version}" advanced \
                      "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"

        -  if test -e "${dirname}/initramfs-${version}-fallback.img" ; then
        -    initrd="${initrd_early} initramfs-${version}-fallback.img"
        -
        -    if test -n "${initrd}" ; then
        -      gettext_printf "Found fallback initrd image(s) in %s:%s\n" "${dirname}" "${initrd_extra} ${initrd}" >&2
        -    fi
        -
        -    linux_entry "${OS}" "${version}" fallback \
        -                "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"
        -  fi
        -
        -  if test -e "${dirname}/booster-${version}.img" ; then
        -    initrd="${initrd_early} booster-${version}.img"
        -
        -    if test -n "${initrd}" ; then
        -      gettext_printf "Found booster initrd image(s) in %s:%s\n" "${dirname}" "${initrd_extra} ${initrd}" >&2
        -    fi
        -
        -    linux_entry "${OS}" "${version}" booster \
        -                "${GRUB_CMDLINE_LINUX} ${GRUB_CMDLINE_LINUX_DEFAULT}"
        -  fi
        -
          if [ "x${GRUB_DISABLE_RECOVERY}" != "xtrue" ]; then
            linux_entry "${OS}" "${version}" recovery \
                        "${GRUB_CMDLINE_LINUX_RECOVERY} ${GRUB_CMDLINE_LINUX}"
        --
        2.51.2
        ```
    3. Generate `grub.cfg` for Arch Linux ALARM as how you do it on a normal installation:
        ```
        mkdir /boot/grub
        grub-mkconfig -o /boot/grub/grub.cfg
        ```
        Note this would not be the first config Grub loads, but rather just a "to-be-included" config.
    4. Add a menuentry in outer Grub config:
        ```
        menuentry 'Arch Linux ARM (aarch64)' {
            configfile ($root)/alarm-aarch64/$suffix
        }
        ```
13. Finalize the installation, umount everything and poweroff

The drive should now work on an aarch64 VM (pure UEFI), but not on real hardware (U-boot faking UEFI), due to missing DTB. Verify it on VM first before trying on real hardware. (Remeber to disable Secure Boot first)

The Grub menu shall look like the following in VM:

```

                         GNU GRUB  version 2:2.14rc1-2

 ┌────────────────────────────────────────────────────────────────────────────┐
 │*Arch Linux (x86_64)                                                        │
 │ Debian (x86_64)                                                            │
 │ Arch Linux ARM (aarch64)                                                   │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘

      Use the ▲ and ▼ keys to select which entry is highlighted.          
      Press enter to boot the selected OS, `e' to edit the commands       
      before booting or `c' for a command-line.                           
                                                                             
```

And after selecting Arch Linux ARM and pressing Enter it's the same old Arch Grub menu (well ALARM didn't bother to modify the look) as always.

Now, for U-Boot to correctly work, U-Boot itself needs to load the specific DTB for device. We don't want Grub to load DTB from root AFTER U-Boot loads Grub, as the device must be fully functional at the time Grub wants to open the Btrfs root, and that's too late.

In modern-day U-Boot, there would be these built-in variables:
- `efi_dtb_prefixes=/ /dtb/ /dtb/current/` -> Same across builds
- `fdtfile=amlogic/meson-sm1-bananapi-m5.dtb` -> Unique for each board
- `scan_dev_for_efi=setenv efi_fdtfile ${fdtfile}; for prefix in ${efi_dtb_prefixes}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${efi_fdtfile}; then run load_efi_dtb; fi;done;run boot_efi_bootmgr;if test -e ${devtype} ${devnum}:${distro_bootpart} efi/boot/bootaa64.efi; then echo Found EFI removable media binary efi/boot/bootaa64.efi; run boot_efi_binary; echo EFI LOAD FAILED: continuing...; fi; setenv efi_fdtfile` -> Same across builds

So take my BPI-M5 for example, the place for the DTB could be:
- `ESP/amlogic/meson-sm1-bananapi-m5.dtb`
- `ESP/dtb/amlogic/meson-sm1-bananapi-m5.dtb`
- `ESP/dtb/current/amlogic/meson-sm1-bananapi-m5.dtb`

I'll pick the second one

For this, we needs to copy some DTBs to "ESP", usually in U-Boot `efi_dtb_prefixes=/ /dtb/ /dtb/current/`, so let's pick `/dtb/` so they can be loaded by U-Boot as early as possible

```
cp -rva /mnt/root/boot/dtbs/linux-aarch64-7ji /mnt/root/efi/dtb
```

And booting on real hardware should now be OK. I've tested this on my BananaPi BPi-M5, OrangePi 5, Orange Pi 5 Plus and they all work seamlessly.

Note: as you may have tried and realized, even if you did not place the DTB, boards could still boot, but in those cases it is the U-Boot's built-in DTB that's used, and for newer kernels this could bring some problems.

Note also: by doing this we've locked the DTB to the one provided by `linux-aarch64-7ji` kernel, which is a stable-as-new-as-possible kernel, **not only for Arch Linux ARM, but also for the latter Debian installation**, using new DTBs on old kernels generally wouldn't bring much trouble, unlike other way around.

## Installing Debian aarch64

Do this in Arch Linux ARM aarch64, just similar to how we installed Debian x86_64 from Arch Linux x86_64. Just remember that still we would not want Grub the boot manager installed here, but rather Debian should only install `grub2-common` to auto-update its sub gurb.cfg

The final Grub menu shall look like this:

```

                         GNU GRUB  version 2:2.14rc1-2

 ┌────────────────────────────────────────────────────────────────────────────┐
 │*Arch Linux (x86_64)                                                        │
 │ Debian (x86_64)                                                            │
 │ Arch Linux ARM (aarch64)                                                   │
 │ Debian (aarch64)                                                           │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘

      Use the ▲ and ▼ keys to select which entry is highlighted.          
      Press enter to boot the selected OS, `e' to edit the commands       
      before booting or `c' for a command-line.                           
                                                                             
```

## After all installation

Now it's good time to do modification, to e.g. decorate the outer menu, reorder the menu, add more entries, etc.

Also note that I did not set timeout, for that booting default x86_64 on aarch64 is plainly wrong. This could be improved by either one of the following ways:
- Use seperate --boot-directory e.g. `/efi/x86_64` and `/efi/aarch64` and maintain sepearate outer grub.cfg for each architecture
- Use `$grub_platform` variable and `cpuid` (x86_64) / `fdtdump` (aarch64) command to determine the current platform, but this is not a guaranteed hit

For simplicity I would keep my current configu without default entry and timout logic.
