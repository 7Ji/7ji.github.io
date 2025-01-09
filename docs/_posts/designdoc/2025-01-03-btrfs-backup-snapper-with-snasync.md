---
layout: post
title:  "Btrfs backup: snapper + snasync"
date:   2025-01-09 11:15:00 +0800
categories: designdoc
---

Usually I don't like to document about my software/script projects on the blog as I prefer to docuemnt them in tree. But as I've been a Btrfs user for multiple years and haven't really documented the details I'd like to share, I think it's a good time to share them together here.

## Background: Btrfs

I'll assume you've already known what Btrfs is: a copy-on-write next-gen Linux filesystem that has entered the kernel for a long time and has many shiny features, of which I appreciate snapshoting and transparent compression the most.

Thanks to Btrfs's nature of copy-on-write, to its helpful userspace toolset btrfs-progs, and to the fact that Btrfs subvolumes live in the filesystem namespace and the user perspective as if they're just folders, it is travail to snapshot a Btrfs filesystem or part of it. 

With the added benefit that a Btrfs subvolume can be mounted directly with specified `subvol=` or `subvolid=` mount argument and their content cannot be snapshotted as a part of the parent folder / subvlume they live in, you can combine your filesystem tree freely to decide which part to take snapshots on.

However, although it is travail to take a snapshot, it is hard to do it cleanly reguarlarly with handwritten script and crontab jobs / systemd.timer units that are easy to forget about and prune to fail. After all, if you've deleted a file by accident you've just created a few hours ago yet your last snapshot are taken a week ago, you're doomed just like you don't have snapshots / backup. 

And of course you would want snapshots to be cleaned reguarly as few would really need a snapshot taken 3 months and 1 hour ago more than a snapshot taken exactly 3 months ago.

## Background: snapper

I'll also assume you've already known what snapper is: a tool written by OpenSUSE to manage filesystem snapshots and allow undo of system modifications. 

To describe it simply: with snapper installed, and its bundled timer units enabled, you can define a few configs that each define what subvolume to take snapshots regurlarly on, and snapper would create and clean up snapshots, and manage them in a centralized way under the corresponding `.snapshots` submount.

Let me take a few lines from my fstab to demonstrate how it works (only the mountpoints of the subvolume to take snapshots and their corresponding snapshots storage subvolume are listed, others, like `/var/cache` are left out):
```conf
# <file system> <dir> <type> <options> <dump> <pass>
## backpane m.2 2280 (2T 4.0 x4 downgraded to 3.0 x4)
ID=nvme-HYV2TBX4_GR__24113WJHA0000072-part3 /                      btrfs rw,compress=zstd:3,subvol=@                       0 0
ID=nvme-HYV2TBX4_GR__24113WJHA0000072-part3 /.snapshots            btrfs rw,compress=zstd:3,subvol=@.snapshots             0 0
ID=nvme-HYV2TBX4_GR__24113WJHA0000072-part3 /home                  btrfs rw,compress=zstd:3,subvol=@home                   0 0
ID=nvme-HYV2TBX4_GR__24113WJHA0000072-part3 /home/.snapshots       btrfs rw,compress=zstd:3,subvol=@home_.snapshots        0 0
## expansion lower m.2 22110 (1T 3.0 x4)
ID=nvme-MZ1LB960HBJR-000FB_S5XBNA0R330322   /srv                   btrfs rw,compress=zstd:15,subvol=@srv                   0 0
ID=nvme-MZ1LB960HBJR-000FB_S5XBNA0R330322   /srv/.snapshots        btrfs rw,compress=zstd:15,subvol=@srv_.snapshots        0 0
## frontpane m.2 2280 (2T 3.0 x4)
ID=nvme-HYV2TBX3_HXY__00000000000000001116  /srv/backup            btrfs rw,compress=zstd:15,subvol=@srv_backup            0 0
ID=nvme-HYV2TBX3_HXY__00000000000000001116  /srv/backup/.snapshots btrfs rw,compress=zstd:15,subvol=@srv_backup_.snapshots 0 0
```
Recall that a Btrfs subvolume's content cannot be snapshotted as a part of the parent folder / subvlume they live in, the reason we have `/`, `/home`, `/srv`, `/srv/backup` as seperate mountpoints is to have them each snapshotted individually into their `.snapshot` subfolder / child subvolume, and `*/.snapshots` are also their seperate subvolumes so their content won't be snapshotted as part of their parent. 

As Btrfs subvolumes can totoally just exist in the FS tree accessible to users as if they're plain folders that just cannot be snapshooted as part of their parent subvolume, it is also possible to omit all these mountpoints and to have a single root subvolume mountpoint, but in that case it would be hard to follow which folder is in fact a subvolume unless you use `btrfs-progs` and corresponding commands. As always I prefer explicity over implicity, so I always set up Btrfs subvolumes directly in the FS root and mount them each individually into the actual root. You can freely decide which way to follow.


I have the following in my `/etc/conf.d/snapper`:
```sh
SNAPPER_CONFIGS="root home srv srv-backup"
```

Are I have the following snapper configs under `/etc/snapper/configs/`:
```sh
> ls -lh /etc/snapper/configs/
total 20K
-rw------- 1 root root 1.2K Jul  3  2024 home
-rw------- 1 root root 1.2K Jul  3  2024 root
-rw------- 1 root root 1.2K Jul 29 16:41 srv
-rw------- 1 root root 1.2K Jan  3 14:22 srv-backup
```

An example config, `home`, is defined as follows:
```
# subvolume to snapshot
SUBVOLUME="/home"

# filesystem type
FSTYPE="btrfs"


# btrfs qgroup for space aware cleanup algorithms
QGROUP=""


# fraction or absolute size of the filesystems space the snapshots may use
SPACE_LIMIT="0.5"

# fraction or absolute size of the filesystems space that should be free
FREE_LIMIT="0.2"


# users and groups allowed to work with config
ALLOW_USERS=""
ALLOW_GROUPS=""

# sync users and groups from ALLOW_USERS and ALLOW_GROUPS to .snapshots
# directory
SYNC_ACL="no"


# start comparing pre- and post-snapshot in background after creating
# post-snapshot
BACKGROUND_COMPARISON="yes"


# run daily number cleanup
NUMBER_CLEANUP="yes"

# limit for number cleanup
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="50"
NUMBER_LIMIT_IMPORTANT="10"


# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="12"
TIMELINE_LIMIT_DAILY="3"
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="6"
TIMELINE_LIMIT_YEARLY="1"


# cleanup empty pre-post-pairs
EMPTY_PRE_POST_CLEANUP="yes"

# limits for empty pre-post-pair cleanup
EMPTY_PRE_POST_MIN_AGE="1800"
```

With the above config, and the bundled `snapper-timeline.timer` and `snapper-cleanup.timer` systemd units enabled, snapper handles the `/home` subvolume as follows:

- Every hour, run `snapper-timeline.service` to take a snapshot of `/home` and store it under `/home/.snapshots/`
  - A new snapshot ID would be generated incrementally, larger than all existing snapshot IDs, e.g. if there are `1`, `2`, `135`, `555`, `705`, then the new ID would be `706`
  - A new folder e.g. `/home/.snapshots/706` would be created as the snapper snapshot container (It's only my own calling, I don't know how snapper calls them)
  - A Btrfs snapshot of subvolume `/home` would be created under the container folder, e.g. `/home/.snapshots/237/snapshot`
  - The corresponding metadata would be stored under the container folder, e.g. `/home/.snapshots/237/info.xml`, with content like the following:

    ```
    <?xml version="1.0"?>
    <snapshot>
      <type>single</type>
      <num>706</num>
      <date>2024-12-22 16:00:05</date>
      <description>timeline</description>
      <cleanup>timeline</cleanup>
      </snapshot>
    ```

- Every hour, run `snapper-cleanup.service` to clean up snapshots under `/home/.snapshots/`
  - As `TIMELINE_LIMITE_HOURLY="12"`, only keep 12 hourly snapshots taken most recently (current, current - 1, ... current - 11; but skip the one taken as 00:00 as it would be considered daily)
  - Likely, only keep 3 daily snapshots taken most recently (skip weekly)
  - Likely, only keep 2 weekly snapshots taken most recently (skip monthly)
  - ...
  - Remove all snapshots that shall not be kept
- If explicitly required, snapshots can be manually created by `snapper create (-c [config])`, e.g. `snapper create -c home`, and manually removed by `snapper delete (-c [config]) [snapshot ID]`, e.g. `snapper delete -c home 1`

With the above setup, the folder `/home/.snapshots` would look like the following after running snapper for a long time:
```
> ls -lh /home/.snapshots/
total 100K
drwxr-xr-x 1 root root 32 Nov 22 18:00 1/
drwxr-xr-x 1 root root 32 Jan  5 00:00 1003/
drwxr-xr-x 1 root root 32 Jan  6 00:00 1027/
drwxr-xr-x 1 root root 32 Jan  7 00:00 1051/
drwxr-xr-x 1 root root 32 Jan  7 12:00 1063/
drwxr-xr-x 1 root root 32 Jan  7 13:00 1064/
drwxr-xr-x 1 root root 32 Jan  7 14:00 1065/
drwxr-xr-x 1 root root 32 Jan  7 15:00 1066/
drwxr-xr-x 1 root root 32 Jan  7 16:00 1067/
drwxr-xr-x 1 root root 32 Jan  7 17:00 1068/
drwxr-xr-x 1 root root 32 Jan  7 18:00 1069/
drwxr-xr-x 1 root root 32 Jan  7 19:00 1070/
drwxr-xr-x 1 root root 32 Jan  7 20:00 1071/
drwxr-xr-x 1 root root 32 Jan  7 21:00 1072/
drwxr-xr-x 1 root root 32 Jan  7 22:00 1073/
drwxr-xr-x 1 root root 32 Jan  7 23:00 1074/
drwxr-xr-x 1 root root 32 Jan  8 00:00 1075/
drwxr-xr-x 1 root root 32 Jan  8 01:00 1076/
drwxr-xr-x 1 root root 32 Jan  8 02:00 1077/
drwxr-xr-x 1 root root 32 Jan  8 03:00 1078/
drwxr-xr-x 1 root root 32 Jan  8 04:00 1079/
drwxr-xr-x 1 root root 32 Jan  8 05:00 1080/
drwxr-xr-x 1 root root 32 Jan  8 06:00 1081/
drwxr-xr-x 1 root root 32 Jan  8 07:00 1082/
drwxr-xr-x 1 root root 32 Jan  8 08:00 1083/
drwxr-xr-x 1 root root 32 Jan  8 09:00 1084/
drwxr-xr-x 1 root root 32 Jan  8 10:00 1085/
drwxr-xr-x 1 root root 32 Jan  8 11:00 1086/
drwxr-xr-x 1 root root 32 Dec  1 00:00 199/
drwxr-xr-x 1 root root 32 Dec 16 17:32 555/
drwxr-xr-x 1 root root 32 Dec 23 00:00 706/
drwxr-xr-x 1 root root 32 Dec 30 00:00 874/
drwxr-xr-x 1 root root 32 Jan  1 00:00 922/
drwxr-xr-x 1 root root 32 Jan  3 10:00 965/
drwxr-xr-x 1 root root 32 Jan  4 00:00 979/
```

An example snapper snapshot container folder `/home/.snapshots/706/` would look like the following:

```
> ls -lh /home/.snapshots/706
total 4.0K
-rw-r--r-- 1 root root 187 Dec 23 00:00 info.xml
drwxr-xr-x 1 root root  36 Aug 26 15:56 snapshot/
```

We can confirm `/home/.snapshots/706/snapshot` is indeed a Btrfs snapshot / read-only subvolume

```
> btrfs subvolume show /home/.snapshots/706/snapshot/
@home_.snapshots/706/snapshot
        Name:                   snapshot
        UUID:                   b54462aa-3427-c745-8fcd-ac248143a039
        Parent UUID:            0ce1faab-e8db-e94f-a3c8-be85743e5859
        Received UUID:          -
        Creation time:          2024-12-23 00:00:05 +0800
        Subvolume ID:           2265
        Generation:             167279
        Gen at creation:        167278
        Parent ID:              259
        Top level ID:           259
        Flags:                  readonly
        Send transid:           0
        Send time:              2024-12-23 00:00:05 +0800
        Receive transid:        0
        Receive time:           -
        Snapshot(s):
        Quota group:            n/a
```

and its content is really what was in `/home` as that point:

```
> ls -alh /home/.snapshots/706/snapshot
total 0
drwxr-xr-x 1 root     root       36 Aug 26 15:56 ./
drwxr-xr-x 1 root     root       32 Dec 23 00:00 ../
drwxr-xr-x 1 root     root        0 Apr  8  2024 .snapshots/
drwx------ 1 nomad7ji nomad7ji 1.3K Dec 22 22:30 nomad7ji/
```

and confirming what we said above, `.snapshots` as a mountpoint would not be recursively snapshotted:

```
> ls -alh /home/.snapshots/706/snapshot/.snapshots
total 0
drwxr-xr-x 1 root root  0 Apr  8  2024 ./
drwxr-xr-x 1 root root 36 Aug 26 15:56 ../
```

A minor thing to note there, is that snapper does not have its "database" that live outside of the subvolume and the container collection, it just calculates what're already been created to decide new things such as new IDs. This is a thing I appreciate, as everything lives inside the snapshot container itself and you do not need extra data or even snapper itself to recover them. But this also has a "side-effect": the ID is not strictly incremental, if you remove e.g. 565, 567, 900, 901, then the new ID could be 565, and if you remove all snapshots then they restart from ID 1 again.

## Data robustness 

Btrfs itself provides you with online data robustness: as long as it runs with CoW feature not turned off then it scrubs and checks data corruption like bitrots reguarlly, and it can report the corruption right away. It's also not easy to be bricked unless you play with RAID56 which is discouraged by Btrfs developers. However, unless you set it up in a Btrfs native RAID1 or alike setup, the corruption cannot be fixed unless you have a backup, as the unique data has only one copy.

snapper enhances the online data robustness by providing the possibility to roll back: it avoids the data loss caused by either accidental, harmful or then-intentional-now-regretful operations. It has saved my ass for multiple times when I accidentally delete my work files, and thankfully I could get the last hourly snapshot to at least restore some of my work. However, as it only snapshots a subvolume on a single filesystem into subvolumes on the same exact filesytem, it is only a hot backup solution and won't do magic if the FS itself breaks.

Both Btrfs itself and snapper lives on the hot, currently active storage, and if the underlying drives dies, they can't have the magic to repair themselves and the data that die along with them.

For complete data robustness you need layered backup, and a solution to do warm/cold backup, which should in most cases be offline. One famous backup strategy that I follow is 3-2-1: to maintain 3 copies of data, to use 2 different types of media, and to keep at least 1 copy off-site.

With the plain Btrfs + snapper setup we already have 2 copies of data: 1 real-time and more than 1 copies in snapshots, I consider the "more than 1 copies in snapshots" part as 1, as the marginal effect decreases fast for the snapshots. What we miss is the last 1 off-site copy of data that's stored on a different type of media.

## snasync

When someone has already defined a good design, it's better to follow it and improve it, rather than throw it away and start all over. As we already have snapper that does the online snapshots creation and cleaning, the best way to have an off-site backup is simply to backup the snapshots to another Btrfs storage so we can have the same robustness as snapper snapshots.

Introducing snasync, a snapper Btrfs snapshots syncer, to backup your Btrfs snapshots created by snapper to warm or cold, and in most cases remote storage.

snasync does one thing and one thing naively: to sync the snapper containers (my own calling, those folders that live under `*/.snapshots` and containing `info.xml` and `snapshot`) one way to "targets": either local, or remote.

As the snapper container ID is only useful to online snapper operations, not strictly incremental, and not meaningful on the timeline perspective, snasync sync local snapper containers to remote in `[prefix]-[timestamp]` name style. An example syncing map is as follows:

```
/home/.snapshots        -> snasync@nas.lan:/srv/backup/snapshots
  - /home/.snapshots/1  ->  - snasync@nas.lan:/srv/backup/snapshots/pc_home-20241101030001
  - /home/.snapshots/2  ->  - snasync@nas.lan:/srv/backup/snapshots/pc_home-20241102030001
  - /home/.snapshots/13 ->  - snasync@nas.lan:/srv/backup/snapshots/pc_home-20241106030001
  ...
```

In which, only `1` would be sent as a whole subvolume, and `2` would be sent as its parent defined as `1`, `13` would be sent as its parent defined as `2`, etc.

Don't worry about the snapper metadata as they're still there:
```
> ls -lh /srv/backup/snapshots/rz5_root-20241122100006/
total 4.0K
-rw-r--r-- 1 root root 185 Jan  3 14:56 info.xml
drwxr-xr-x 1 root root 150 Jan  3 14:56 snapshot/
> cat /srv/backup/snapshots/rz5_root-20241122100006/info.xml
<?xml version="1.0"?>
<snapshot>
  <type>single</type>
  <num>1</num>
  <date>2024-11-22 10:00:06</date>
  <description>timeline</description>
  <cleanup>timeline</cleanup>
</snapshot>
```

The timestamp is extracted from the metadata and explicited included in the name so it's easier to look up without the help of snapper.

An example snasync run is as follows:
```bash
targets=(
    --target /srv/backup/snapshots # a simple path refers to local target, this is my online warm backup
    --target snasync@wtr.fuo.lan:/srv/backup/snapshots # a scp-style target means to sync to remote, this is my off-site cool backup (not entirely cold as not off-line)
)
snasync \
    --source / --prefix rz5_root "${targets[@]}" \
    --source /home --prefix rz5_home "${targets[@]}" \
    --source /srv --prefix rz5_srv "${targets[@]}"
```

Basically you define a `source` to sync from (either the subvolume containing `.snapshots`, or `.snapshots` itself, snasync would figure it out), and the prefix to name the synced snapshots with (otherwise would be derived from source path), and then a few targets to sync them to, each source has their own targets, but in this case they're all the same.

The whole operating logic of snasync is as follows:
- If `snapper-cleanup.timer` is running, stop it and register an on-exit trigger to start it again.
- If `snapper-cleanup.service` is running, wait for it to finish.
- Scan for all sources to get a list of snapper containers to sync, the list would not be updated during this run again are is essentially read-only.
  - If a snapshot is read-write, skip it
  - If `info.xml` is missing, skip it
  - Timestamp is extracted from `info.xml` and used as key
  - A timestamp-name map, a timestamp-path map are each created with timestamp with key
  - Timestamps are sorted so later operations go from the minimum to the maximum
- Iterate through all targets to extract the list of remotes, and do simple SSH connection to each of them with control master specified and quit to warm up the connection. 
  - If a remote is out of reach, mark it as bad, and it would not be used in later syncing
- For each source, fork out to sync it so we're syncing with multi-process
- In the source syncer, for each target, for out to sync it so we're syncing with multi-process
- Iterate through the corresponding snapper containers, for each of them:
  - If a snapshot was already synced, define parent as the last snapshot, otherwise keep it empty
  - If a container exist in target:
    - If `snapshot` is read-only or contains received-UUID:
      - If `info.xml` does not exist, copy the source one to it
      - Consider this container already synced and skip to the next one
    - If `snapshot` is missing, or not read-only, or does not contain received-UUUID:
      - Delete the `snapshot` and `info.xml`
  - If a container does not exist in target, create the container
  - Sync the source `snapshot` to target
    - Sender has argument `--compressed-data` to keep the compressed data to save bandwidth, even if it's running a local sync
    - Sender has argument `--parent [snapshot]` if we have already seen synced snapshot to do only incremental sync, otherwise it does not have such argument
    - Receiver has argument `--force-decompress` to decompress the data first and then compress with the target compression options
- Iterate through target containers that start with the prefix and named in expected format
  - If it's seen at source, keep it untouched
  - If it's not seen at source, rename it to add `.orphan` suffix. It's up to users whether to delete or keep them.
- In the source syncer, collect target syncers
- In the main worker, collect source syncers
- Bring back `snapper-cleanup.timer` if this was registered

On my remote backup server the layout looks like the following:
```
> ls -l /srv/backup/snapshots/
total 0
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241126130020/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241224130016/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241224140003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241225160003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241225170003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241226130022/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241226140022/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241227160003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241227170003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241227180003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241228150001/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241228160001/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_home-20241228170003/
drwxr-xr-x 1 root root 32 Dec 29 17:55 dsk_root-20241126130020/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241224130016/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241224140003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241225160003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241225170003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241226130022/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241226140022/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241227160003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241227170003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241227180003/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241228150001/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241228160001/
drwxr-xr-x 1 root root 32 Dec 29 17:57 dsk_root-20241228170003/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20240920170000/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20240930160000/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241031160001/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241130160020/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241208160007/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241215160017/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241217160023.orphan/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241218160025.orphan/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241219160025/
drwxr-xr-x 1 root root 32 Dec 23 21:36 fuo_home-20241220160005/
......
drwxr-xr-x 1 root root 32 Dec 24 14:48 wtr_root-20241224040006/
drwxr-xr-x 1 root root 32 Dec 24 14:48 wtr_root-20241224050016/
drwxr-xr-x 1 root root 32 Dec 24 14:48 wtr_root-20241224060000/
```

As the snapshots are sent and received with parent if possible, and compressed on target which shall be mounted with higher compression level than source, the disk space they take is not very much:
```
> sudo btrfs filesystem du -s /srv/backup/snapshots/*
     Total   Exclusive  Set shared  Filename
  25.81GiB    28.26MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241126130020
  25.80GiB     4.57MiB    18.97GiB  /srv/backup/snapshots/dsk_home-20241224130016
  25.79GiB     7.00MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241224140003
  25.79GiB     1.47MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241225160003
  25.79GiB     1.57MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241225170003
  25.79GiB     3.44MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241226130022
  25.79GiB     4.17MiB    18.96GiB  /srv/backup/snapshots/dsk_home-20241226140022
  ...
```

Note the snasync only renames the containers that do not exist locally to have a `.orphan` suffix but not delete them, their content, `snapshot` and `info.xml` are still there:
```
> ls -lh /srv/backup/snapshots/fuo_home-20241217160023.orphan/
total 4.0K
-rw-r--r-- 1 root root 188 Dec 23 21:43 info.xml
drwxr-xr-x 1 root root  36 Jul  9  2024 snapshot/
```

One can remove all orhpaned containers and snapshots in them if they want to free up space:
```
> sudo btrfs subvolume delete /srv/backup/snapshots/*.orphan/snapshot
> sudo rm -rf /srv/backup/snapshots/*.orphan
```

When backed up data is needed one can simply navigate through all the containers and snapshots
```
nomad7ji@wtr /s/b/snapshots> cd rz5_home-20241122100006/
nomad7ji@wtr /s/b/s/rz5_home-20241122100006> ls
info.xml  snapshot/
nomad7ji@wtr /s/b/s/rz5_home-20241122100006> cd snapshot/
nomad7ji@wtr /s/b/s/r/snapshot> ls
nomad7ji/
nomad7ji@wtr /s/b/s/r/snapshot> cd nomad7ji/
nomad7ji@wtr /s/b/s/r/s/nomad7ji> ls
Android/  Building/  Desktop/  Development/  Documents/  Downloads/  go/  Music/  opt/  Pictures/  Public/  Security/  Templates/  Videos/
```

## Offline cold backup is still needed

While Btrfs + snapper + snasync setup provides enough robustness, all of them relies on the robustness of Btrfs filesystem, and it's not impossible to fail, and you can not trust the underlying storage 100%.

When possible, please do regular offline cold backup. I do yearly BD-R backups with 25GB HTL BD-R disces with 10% parity volume, and hoard the disces at my parents' and the parity volumes on network drive. It does not matter if the data is hard to retrieve, so it's even OK to store it on e.g. AWS S3. They're only needed when everything else fails, and at that point, every part of the work to retrieve them are always worth it.