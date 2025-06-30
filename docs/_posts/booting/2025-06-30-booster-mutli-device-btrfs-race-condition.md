---
layout: post
title:  "Booster's multi-device Btrfs race condition"
date:   2025-06-30 11:50:00 +0800
categories: booting
---

_Repost of booster [Pull Request #299](https://github.com/anatol/booster/pull/299) created by myself_:

In a recent change to kernel pacakage the module for Btrfs became built-in instead of as-module: [diff](https://gitlab.archlinux.org/archlinux/packaging/packages/linux/-/commit/a7e2a17f9c0e55937ea3e18c4d5b905a8e4f8047)

This reveals a race condition (in both kernel and booster) which seemed to have been "worked around" in the past but in reality not, for multi-device Btrfs:
- Booster did not check the return status of `BTRFS_IOC_DEVICES_READY` and it mounts every Btrfs sub-device whether they're actually ready or not.
- The kernel considers a Btrfs filesytem being "used" while it is being mounted from sub-device A and would refuse to mount it from sub-device B, even though it would fail to mount A after only a short time window.
- The logic was:
  - Goroutine A:
    1. "booster sends IOCTL to register sub-device A"
    2. "booster always think sub-device A ready" (definitely not ready)
    3. "booster mounts sub-device A"
  - Go routine B:
    1. "booster sends IOCTL to register sub-device B"
    2. "booster always think sub-device B ready" (maybe ready)
    3. "booster mounts sub-device B"
- When btrfs is a module, when "booster always think sub-device B ready", it is really ready in most cases; when btrfs is built-in instead of a module, all of the above happen too fast and the failure of mounting from A was not finished yet, so the step "booster sends IOCTL to register sub-device B" ends up being rejected by btrfs kernel module with `EBUSY`

Example logs of failed boots, from which you could tell that booster tries to register `vdc` immediately after `vdb` while `vdb`'s failure was not finished yet and it's rejected by kernel:

```
[    0.875882] BTRFS error: device /dev/vdc (254:32) belongs to fsid f1872b6c-c2b3-44fe-930e-bb85fd35d669, and the fs is already mounted, scanned by init (1)
[    0.876896] booster: ioctl(0x90009427): device or resource busy
ioctl(0x90009427): device or resource busy
[    0.877787] BTRFS error (device vdb): devid 2 uuid 1f95bd95-1c49-498f-abf9-a39143b970ce is missing
[    0.878568] BTRFS error (device vdb): failed to read the system array: -2
[    0.880541] BTRFS error (device vdb): open_ctree failed: -2
mount(/dev/vdb): no such file or directory
```

```
found a new device /dev/vda
blkinfo for /dev/vda: type=mbr UUID=dfd9de13 LABEL=
found a new device /dev/vda1
blkinfo for /dev/vda1: type=fat UUID=8b5649c8 LABEL=NO NAME
found a new device /dev/vdb
blkinfo for /dev/vdb: type=btrfs UUID=f1872b6c-c2b3-44fe-930e-bb85fd35d669 LABEL=
mounting /dev/vdb->/booster.root, fs=btrfs, flags=0x0, options=
found a new device /dev/vdc
blkinfo for /dev/vd[    0.833072] BTRFS error: device /dev/vdc (254:32) belongs to fsid f1872b6c-c2b3-44fe-930e-bb85fd35d669, and the fs is already mounted, scanned by init (136)
c: type=btrfs UU[    0.834106] booster: ioctl(0x90009427): device or resource busy
ID=f1872b6c-c2b3[    0.834346] BTRFS error (device vdb): devid 2 uuid 1f95bd95-1c49-498f-abf9-a39143b970ce is missing
-44fe-930e-bb85f[    0.835248] BTRFS error (device vdb): failed to read the system array: -2
d35d669 LABEL=
ioctl(0x90009427): device or res[    0.836022] BTRFS error (device vdb): open_ctree failed: -2
ource busy
mount(/dev/vdb): no such file or directory
```

To fix this, we need to check the return status of `BTRFS_IOC_DEVICES_READY` and only really considers the FS ready when it returns 0

Now the logic is:
- Goroutine A:
    1. "booster sends IOCTL to register sub-device A"
    2. "booster knows sub-device A not ready and waits" -> back to step 1
    3. No bad mounting of sub-device A is performed
- Go routine B:
    1. "booster sends IOCTL to register sub-device B"
    2. "booster knows sub-device B ready" (definitely ready)
    3. "booster mounts sub-device B"

Tested on a VM with three Vdisks, vda as boot, vdb + vdc as btrfs (profile single, raid0, raid1 all tested):
```
found a new device /dev/vda
blkinfo for /dev/vda: type=mbr UUID=dfd9de13 LABEL=
found a new device /dev/vdc
found a new device /dev/vda1
blkinfo for /dev/vdc: type=btrfs UUID=f1872b6c-c2b3-44fe-930e-bb85fd35d669 LABEL=
blkinfo for /dev/vda1: type=fat UUID=8b5649c8 LABEL=NO NAME
found a new device /dev/vdb
Waiting for multi-device btrfs at /dev/vdc to become fullly assembled, waited 0 seconds
blkinfo for /dev/vdb: type=btrfs UUID=f1872b6c-c2b3-44fe-930e-bb85fd35d669 LABEL=
mounting /dev/vdb->/booster.root, fs=btrfs, flags=0x0, options=
Switching to the new userspace now. Да пабачэння!
```

References:
 - btrfs kernel module returns 0 for ready and 1 for not ready to `BTRFS_IOC_DEVICES_READY`: [source code](https://github.com/torvalds/linux/blob/86731a2a651e58953fc949573895f2fa6d456841/fs/btrfs/super.c#L2247)
 - btrfs-progs checks the return value of `BTRFS_IOC_DEVICES_READY` with type `int` to determine whether the FS is ready: [source code](https://github.com/kdave/btrfs-progs/blob/5d47f58fc37fbc93d630d30183e8a2d3354d58e6/cmds/device.c#L530)
