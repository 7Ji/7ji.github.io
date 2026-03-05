---
layout: post
title:  "Into Alpine APK v3 format: the binary perspective"
date:   2026-03-03 11:30:00 +0800
categories: designdoc
---

## Background

From OpenWrt 25.12 onwards, and available earlier in release candidates and development branches, OpenWrt has swapped their package manager + package format from `opkg` + `.ipk` (almost a `.deb`) to  `apk-tools v3.0` from Alpine Linux and its newly introduced `APK v3`. Even Alpine itself has just upgraded `apk-tools` to `v3.0`  since `v3.23` but not fully switched into `apk v3` yet.

The format is a totally new format designed in-house by Alpine with some collabration from OpenWrt. But let's first list the above mentioned older formats:

- `deb` is an `ar` like format with pre-defined members `debian-binary`, `data.tar(.gz/.xxx)`, `control.tar(.gz/.xxx)`, the files are in `data.tar.xxx` and metadata is in `control.tar.xxx`
- `ipk` is pretty much a fixed version `deb` with in most cases only gzip compressed `data` and `control` so less dependency is needed which is useful in embedded scenarios.
- `apk` v2 is three (with signature) or two gzipped (without signature) `tar`s, one optionally for signature, one for metadata (kinda like `control.tar` in `deb`) and one for actual files (kinda like `data.tar` in deb)

`apk` v3 however is in a supposedly schema-based binary format `adb`, loosely defined by its official manual ([source](https://github.com/alpinelinux/apk-tools/blob/master/doc/apk-v3.5.scd) / [online](https://man.archlinux.org/man/extra/apk-tools/apk-v3.5.en)), yet most details are determined by `apk-tools` the only reference tool's C source code.

I've recently wrote a tool [adumpk](https://github.com/7Ji/adumpk/) to parse an v3 `.apk`, prints useful info about it, optionally convert it to `.tar`, and write metainfo into `.json`. When writing the tool I had to jump around in `apk-tools`'s source code quite often. After finishing adumpk, I decided to write this as an easier-to-follow single blog post so others can avoid the hassle.

## Format

Unless explicitly mentioned, all integral data types in the format is little-endian.

The below uses [openwrt/25.12.0-rc5/crowdsec-1.6.2-r1.apk](http://downloads.openwrt.org/releases/25.12.0-rc5/packages/x86_64/packages/crowdsec-1.6.2-r1.apk) as an example package, although it's not needed, it's recommended you get one so it's easier to examine the binary by yourself and learn the actual binary format.

### File Header

The file header is 4 bytes in length, first 3 bytes being magic `"ADB"`, i.e. big-endian `0x616462`, then the last byte is either one of the following to tell the file compression method:

- `.`, i.e. `0x2e`, means the content is not compressed
- `d`, i.e. `0x64`, means the conetnt is compressed with Deflate level 0
- `c`, i.e. `0x63`, means the content is compressed with a custom compression method recorded in the next 2 bytes
  - the first byte is a `u8` recording the ID of compression method:
    - 0 for not compressed
    - 1 for Deflate
    - 2 for Zstandard, which is optionally supported only when `apk-tools` was compiled with `HAVE_ZSTD`
  - the second byte is a `u8` containing compression level, the level allowed by each of the above method is:
    - not compressed: 0
    - Deflate: 0 to 9
    - Zstandard: 0 to 22

### Compressed Body

The compressed `ADB` data body either starts from the 7th byte (`file[6:]`) if the compression method is `c` and 2 addtional bytes were took to record the actual compression method, or the 5th byte (`file[4:]`) otherwise.

The compressed `ADB` data body shall be **a plain stream without magic header** you would expect from a standalone compressed file, so the gzip stream does not begin with big-endian `0x1F8B` magic then meta, and zstd stream does not begin with big-endian `0x28B52FFD` magic.

Note while Deflate stream can be concatenated to each other, which is used in `apk v2`, the `apk v3` body should be a whole continous Deflate stream if it was compressed with such algorithm. _I didn't test this but I think the official tool would happily accept a manually prepared `apk v3` with multiple concatenated Deflate stream_. **Zstandard on the other hand does not support concatenated streams.**

The body, when decompressed, shall begin with exactly `adb.`, same as uncompressed `apk v3`, 3-byte magic and 1-byte marking no compression.

With the example apk the body shall be decompressed with:

```python
import zlib
with open('crowdsec-1.6.2-r1.apk', 'rb') as f:
    f.read(4)
    body=zlib.decompress(f.read(), wbits=-15)
```

### Top-level schema

After the leading `adb.` in either the original uncompressed file or the decompresssed body, a 4-byte (`u32(body[4:8])`) magic (called `schema`) would mark the inner data as one of the following:

- `0x676B6370` or big-endian `0x70636B67` or literal `pckg` for package
- `0x78646E69` or big-endian `0x696E6478` or literal `indx` for index

We would only talk about package.

### ADB blocks stream

In the package case, a series of `ADB block`s would continue one after another, each starting at a 8-byte boundary (the first `ADB block` naturally starts at such boundary as it's after 8-byte `"ADB.pckg"`).

Each block starts with a `u32` recording the type and optionally the size, itself as either a simple header, or as the first member of a bigger 16-byte header

- If the highest 2 bits are not all `1` (so either `0b00`, `0b01`, `0b10`, but not `0b11`), then this is a simple 4-byte header
  - Type is these 2 bits extracted, i.e. `v >> 30`
  - Raw size for this block (4-byte header included) is the low 30 bits, i.e. `v & 0x3fffffff`
- If the highest 2 bits are all `1`, then this is an extended 16-byte header, including the 4-byte u32 `v` itself as `type_size` field, a 4-byte u32 reserved field for alignment for future expansion, and a 8-byte u64 `x_size` field, defined in C as:
  ```c
  struct adb_block {
      uint32_t type_size;
      uint32_t reserved;
      uint64_t x_size;
  };
  ```
  - Type is the low 30 bits, `i.e. u32 & 0x3fffffff`
  - Raw size for this block (16-byte header included) is the `x_size` field

The payload size of what follows the header can be calculated as `raw size - header size`, and it must be non-negative.

The actual type of a block must be one of the following:

|ID|NAME|Usage|
|-|-|-|
|0|ADB_BLOCK_ADB|essential, contains metadata info and file infos|
|1|ADB_BLOCK_SIG|optional, carries signature data|
|2|ADB_BLOCK_DATA|technically also optional, carries file content (no name or path) or alike|

The order of these blocks is restricted. A sane `ADB` must contains these blocks head to tail:

- 1 `ADB_BLOCK_ADB` for metadata
- 0 to N `ADB_BLOCK_SIG` for signature
- 0 to N `ADB_BLOCK_DATA` for data

If the order is not respected (e.g. `SIG` after `DATA`, or `ADB` after `SIG`/`DATA`, etc ), then the file would be rejected

### The meta block `ADB_BLOCK_ADB`

Such block must be the first block in one package's blocks stream.

The block begins with a 8-byte header, including a u8 `adb_compat_ver` field (currently must be `0`), a u8 `adb_ver` field (currently also must be `0`), a u16 `reserved` field for alignment and future expansion (all 0), then a u32 `root` field to declare the following data streams, type aliased as `adb_val_t`, defined in C as:

```c
struct adb_hdr {
    uint8_t adb_compat_ver;
    uint8_t adb_ver;
    uint16_t reserved;
    adb_val_t root;
};
```

An `adb_val_t` `val` carries type info in its highest 4 bits (`val & 0xf0000000`), and value in lowest 28 bits (`val & 0x0fffffff`)

Let's focus on what `root` means here.

The type shall be `ADB_TYPE_OBJECT` (`0xe0000000 == root & 0xf0000000`) for a series of elements each with their own types, which makes sense for a metadata block.

And the value, i.e. `root & 0x0fffffff`, marks the offset of targets **inside current payload**, note the current payload includes the above `adb_hdr` but not the block header, so e.g. with `u32([body[8:12]]) == 0x1614 == 5652` for a simple 4-byte header, type `ADB`, this means the whole block is at `body[8:8+5652] == body[8:5660]`, so payload is `body[8+4:5660] == body[12:12+5648]` without the block header, including the `adb_hdr` at `body[12:12+8] == body[12:20]`, so with `root==0xe0001600` the offset would be `0x1600 == 5632`, and therefore we need to go from `body[12+5632]`.

### ADB Object for root

The pointed-to `ADB` object elements for root starts with a u32 recording how many `adb_val_t`s follow it **including itself**, then the actual series of `adb_val_t`s, so e.g. with the above example and `u32(body[5644:5648])` being 4, then the 3 `adb_val_t`s are expected at `body[5648:5652]`, `body[5652:5656]`, `body[5656:5660]`.

_Note the last `adb_val_t` is just at the end of this payload / ADB\_BLOCK_ADB, you can easily tell that the `root` value was written at the end of `ADB_BLOCK_ADB` creation_

The above 3 objects with ID starting at 1, alongside count u32 (`4`) with ID 0 (remember this convention, APK prefers to use id 0 for num/count and 1 onwards as actual slots), can be considered a 4-length `adb_val_t` array, each latter member is used for a different purpose:

|ID|NAME|Purpose|
|-|-|-|
|1|ADBI_PKG_PKGINFO|package info metadata|
|2|ADBI_PKG_PATHS|package file / folder paths (compacted than plain list of texts)|
|3|ADBI_PKG_SCRIPTS|package postrm / preinst / etc scripts|
|4|ADBI_PKG_TRIGGERS|package triggers|

If a slot is not needed then it can be 0 if there's still slot needed after it, or not exist at all if there's no slot needed after it. In this example with count being 4 there would only be 3 slots, so no `TRIGGERS` was stored.

We would only focus on `PKGINFO` and `PATHS`, as `SCRIPTS` and `TRIGGERS` pretty much follows the idea of `PATHS` + `ADB_BLOCK_DATA`

### ADBI_PKG_PKGINFO: Package Info

ID 1 in root object, which marks the head of pacakge info, is yet another object, e.g. `root_obj[1] == body[5648:5652] ==  0xe00012b8`, means an object (`0xe00012b8 & 0xf0000000 == 0xe0000000`) with offset 4792 (`0xe00012b8 & 0x0fffffff == 0x12b8 == 4792`), which then points to another `u32` for number for elements including itself. So e.g. `u32(body[12+4792:+4]) == u32(body[4804:4808]) == 17` means there're 16 `adb_val_t` after the count u32, 1st at `body[4804+4*1:+4] == body[4808:4812]` and 16th at `body[4804+4*16:+4] == body[4868:4872]`.

Now is a good time to list all possible data types, which can all be possibly used in these fields:

|Type|Magic|Note|
|-|-|-|
|ADB_TYPE_SPECIAL|0x00000000|Currently just alias to `INT`|
|ADB_TYPE_INT|0x10000000|Single u32 (max 0x0fffffff) value in low|
|ADB_TYPE_INT_32|0x20000000|Single u32 at low-as-off|
|ADB_TYPE_INT_64|0x30000000|Single u32 at low-as-off|
|ADB_TYPE_BLOB_8|0x80000000|Series of u8, length (u8) + data (u8s) at low-as-off|
|ADB_TYPE_BLOB_16|0x90000000|Series of u16, length (u16) + data (u16s) at low-as-off|
|ADB_TYPE_BLOB_32|0xa0000000|Series of u32, length (u32) + data (u32s) at low-as-off|
|ADB_TYPE_ARRAY|0xd0000000|Series of same type, length (u32) + data at low-as-off|
|ADB_TYPE_OBJECT|0xe0000000|Series of different type, length (u32) + data at low-as-off|

As we briefly mentioned earlier, as each element does not record its own ID yet the ID has special meaning, to mark an empty, skipped element, the elements shall be special value 0; e.g. if a package has only a ID 6 field to declare, then it would have first 5 slots all set to 0 so they're skipped, and there would be no ID 7 field onwards.

These slots are numbered as follows:

|ID|Name|Data Type|
|-|-|-|
|1|NAME|BLOB_8, as string|
|2|VERSION|BLOB_8, as string|
|3|HASHES|BLOB_8, as hex-string|
|4|DESCRIPTION|BLOB_8, as string|
|5|ARCH|BLOB_8, as string|
|6|LICENSE|BLOB_8, as string|
|7|ORIGIN|BLOB_8, as string|
|8|MAINTAINER|BLOB_8, as string|
|9|URL|BLOB_8, as string|
|10|REPO_COMMIT|BLOB_8, as hex-string|
|11|BUILD_TIME|INT|
|12|INSTALLED_SIZE|INT|
|13|FILE_SIZE|INT|
|14|PROVIDER_PRIORITY|INT|
|15|DEPENDS|OBJECT of dependency (see below)|
|16|PROVIDES|OBJECT of dependency (see below)|
|17|REPLACES|OBJECT of dependency (see below)|
|18|INSTALL_IF|OBJECT of dependency (see below)|
|19|RECOMMENDS|OBJECT of dependency (see below)|
|20|LAYER|INT|
|21|TAGS|OBJECT of BLOB_8, as string array|

So e.g. the first element, `u32(body[4808:4812]) == 0x80000008`, is not zero, means the package has an actual name, the high `0x8` means this is a `BLOB_8` item, and the low `0x8` means the item's length is at offset 8 and content starts at offset 9, so, `length == u8(body[12+8]) == 8`, and therefore content is at `body[12+9:12+9+8] == body[21:29]`, the example package is `crowdsec`

And e.g. the last element ID 16 `u32(body[4868:4872]) == 0xe0000184`, is not zero, means the package has an actual `provides` `OBJECT` (`0xe0...`), offset `0x184 == 388` in payload, so `u32(body[12+388:12+388+4]) == u32(body[400:404]) == 2` records the number of sub-elements including the count itself, therefore 1 actual element, `adb_val_t(body[404:408])` therefore records the info of the only sub-element, here being `0xe000017c` means it's yet another `OBJECT` starting at offset `0x17c`, ..., and then `u32(body[392:396]) == 2` so there's again 1 real sub-element, `adb_val_t(body[396:400]) == 0x8000016c` means this is a BLOB8 starting at `0x16c`, then we get length from `u8(body[12+0x16c]) == 12`, and finally the provide item at `body[12+0x16c+1:+12] == body[377:389]`, being `crowdsec-any`

While the `provides` seems a list of list of BLOB8, but recall that `OBJECT` elements can be different types, each element in `provides` is actually a strongly-typed dep info, containing the `NAME` slot (BLOB8, ID1), `VERSION` slot (BLOB8, ID2), and `MATCH` slot (INT, ID3, for vercmp operations). In the example there's just no `VERSION` nor `MATCH`.

All dependency-like element can have these 3 slots:

|ID|Name|Data Type|
|-|-|-|
|1|NAME|BLOB8, string|
|2|VERSION|BLOB8, string|
|3|MATCH|INT|

The `MATCH` field is a bitwise OR of the following base bits:

|NAME|VALUE|bit|
|-|-|-|
|EQUAL|1|0b00001|
|LESS|2|0b00010|
|GREATER|4|0b00100|
|FUZZY|8|0b01000|
|CONFLICT|16|0b10000|

The implementation would pre-calculate all **valid** combinations of these; these are (excluding `CONFLICT` which can be freely appended):

|Sign|Meaning|Bits|
|-|-|
|<|Less than|0b0010|
|<=|Less than or equal to|0b0011|
|<~|less than or equal to, fuzzy|0b1011|
|~|Fuzzy|0b1000|
|=|Equal to|0b0001|
|>=|Greater than or equal to|0b0101|
|>~|Greater than or equal to, fuzzy|0b1101|
|>|Greater than|0b0100|
|><|Special, checksum|0b0110|
|_|Special, any|0b0111|

So e.g. a match field with value `0x10000003` would mean `INT` with value `0x3 == 0b11`, therefore `<=`

### ADBI_PKG_PATHS: Paths

Files and folders are stored in `ADB_BLOCK_ADB` in a compact way, before the latter possible file data appearance in `ADB_BLOCK_DATA` blocks. Each of these path elements stores a folder path without the leading / (_empty path for root folder_), then any number of direct file entries. While most of the file entries do need their `ADB_BLOCK_DATA` block for actual data, others could exist purely in header.

ID 1 in `ADB_BLOCK_ADB`'s root object, which marks the head of paths, is yet another object, e.g. `root_obj[2] == body[5652:5656] ==  0xe00012fc`, means an object with offset 0x12fc, and `u32(body[12+0x12fc:+4]) == u32(body[4872:4876]) == 22` means there're 21 `adb_val_t` after the count u32, 1st at `body[4872+4*1:+4] == body[4876:4880]` and 21st at `body[4872+4*21:+4] == body[4956:4960]`

Each one of these 21 "path"s is actually called `ADBI_DI` by `apk-tools`, and is also an `OBJECT` with the following slots (still, some are optional):

|ID|NAME|Data Type|
|-|-|-|
|1|NAME|BLOB8, string|
|2|ACL|OBJECT being ACL info (see below)|
|3|FILES|OBJECT of File info (see below)|

In the example the last "path" `adb_val_t(body[4956:4960]) == 0xe0001290`, so it's an `OBJECT` starting from `12 + 0x1290 == 4764`, as `u32(body[4764:4768]) == 4` there're 3 slots after it.

For ID 1, NAME, `adb_val_t(body[4768:4772]) == 0x80001258`, it's a `BLOB8` starting at offset `0x1258`, and `u8(body[12+0x1258]) == u8(body[4708]) == 7` says this is a 7-length string, content at `body[4708+1:7] == body[4709:4716] == b"usr/bin"` says the folder name/path is `usr/bin`

For ID2, ACL, `adb_val_t(body[4772:4776]) == 0xe0000194`, it's an `OBJECT` starting at offset `0x194`, the count `u32(body[12+0x194:+4]) == u32(body[416:420]) == 4` so there're 3 slots after it.

The ACL info OBJECT could have the following slots:

|ID|NAME|Data Type|
|-|-|-|
|1|MODE|INT|
|2|USER|BLOB8, string|
|3|GROUP|BLOB8, string|
|4|XATTRS|OBJECT of BLOB8, each BLOB8 with `\0` as sep for name and value|

In the example:
- SLOT1 reads `0x100001ed` so it's an `INT` with value `0x1ed == 0o755`
- SLOT2 reads `0x8000018c` so it's an `BLOB8` starting at offset `0x18c`, length at `body[408]` reads `4` and content at `body[409:413]` reads `root`
- SLOT3 reads the same value so it reuses `root` from `USER`
- There's not SLOT4

For ID3, FILES, `adb_val_t(body[4776:4780]) == 0xe0001260`, it's an `OBJECT` starting at offset `0x1260`, the count `u32(body[12+0x1260:+4]) == u32(body[4716:4720]) == 12`, so there're 11 file entries after it, the first at `body[4720:4724]` and the last at `body[4760:4764]`.

The first file entry, `adb_val_t(body[4720:4724]) == 0xe0000f4c`, it's an `OBJECT` starting at offset `0xf4c`, the count `u32(body[12+0xf4c:+4]) == u32(body[3928:3932]) == 6`, so there're 5 slots after it.

The File info OBJECT could have the following slots:

|ID|NAME|Data Type|
|-|-|-|
|1|NAME|BLOB8, string|
|2|ACL|OBJECT being ACL info (see above)|
|3|SIZE|INT|
|4|MTIME|INT32|
|5|HASHES|BLOB8, hex-string|
|6|TARGET|BLOB8, string|

In the example:
- SLOT1 reads `0x80000008` so it's an `BLOB8` with length at offset 8, `u8(body[12+8]) == 8`, and content `body[12+8+1:+8] == b"crowdsec"`
- SLOT2 reads `0xe0000194` so it's again `0o755` owned by `root:root`
- SLOT3 reads `0x133cd7e8` so it's an `INT` with value `0x33cd7e8 == 54319080`
- SLOT4 reads `0x200001e4` so it's an `INT_32` at offset `0x1e4`, and `u32(body[12+0x1e4:+4]) == 1772344484` so mtime is `Sun Mar  1 05:54:44 UTC 2026`
- SLOT5 reads `0x80000f28` so it's an `BLOB8` with length at offset 0xf28, `u8(body[12+0xf28]) == 32`, and content `body[12+0xf28+1:+32]` reads a hex-string, which is the SHA256 checksum of the file
- There's no SLOT6, as this is a regular file

A file can have its `SIZE` set to 0, being an empty file, and on top of it having `TARGET` set, so it either serves as a symlink or hardlink to the set target, or is a special `CHAR` / `DEV`.

Let's use the third file entry under the same last path entry to examine what `TARGET` does, which is `adb_val_t(body[4728:4732]) == 0xe0000fdc`, it's an OBJECT with offset 0xfdc, we read `u32(body[12+0xfdc:+4]) == u32(body[4072:4076]) == 7` so it does have 6th slot for `TARGET`; we read the name `adb_val_t(body[4076:4090]) == 0x80000fac` so name starts at offset 0xfac and `len == u8(body[12+0xfac]) == u8(body[4024]) == 5`, content is `body[4024+1:+5] == body[4025:4030] == b"cscli"`, so the link itself is `usr/bin/cscli`; we skip to SLOT 6 for `TARGET` which should be `adb_val_t(body[12 + 0xfdc + 4 * 6:+4]) == adb_val_t(body[4096:4100]) == 0x80000fb2`, so it's a BLOB with offset `0xfb2`, then we read length at `u8(body[12+0xfb2]) == u8(body[4030]) == 23` so content is `body[4030+1:+23] == body[4031:4054] == b"\x00\xa0/usr/bin/crowdsec-cli"`

The first two bytes in the `TARGET` determines the data type, and they shall be handled as one u16, and `u16(body[4031:4033]) == 40960 == 0o120000`, this is basically the same thing as you would expect from the `st_mode` field in a `struct stat` with already `S_IFMT` been bitwise AND. The following file type are supported:

|Type|Mask|Content at target[2:]|
|-|-|-|
|S_IFBLK|0o060000|8-byte, as u64 for dev ID (major:minor combined)|
|S_IFCHR|0o020000|8-byte, as u64 for dev ID (major:minor combined)|
|S_IFIFO|0o010000|8-byte, as u64 for dev ID (major:minor combined)|
|S_IFLNK|0o120000|any-length, for symlink target|
|S_IFREG|0o100000|any-length, for hardlink target|

The real target for symlink is therefore `target[2:]`, so we know this symlink is `usr/bin/csli -> /usr/bin/crowdsec-cli`

**When reading through the `PATH`s info it's recommended to store them for later lookup, as the `ADB_BLOCK_DATA` blocks would only carry the file content, but not the names, paths, ownership, etc.**

### The signature block `ADB_BLOCK_SIG`

Such block must be after `ADB_BLOCK_ADB` and before `ADB_BLOCK_DATA`.

The block begins with a 2-byte header, including a u8 `sign_ver` field for the version of signature (currently must be 0), and a u8 `hash_alg` field for the ID of the algorithm, defined in C as:

```c
struct adb_sign_hdr {
    uint8_t sign_ver, hash_alg;
};
```

The hash algorithm could be one of the following:

|ID|NAME|LENGTH|
|-|-|-|
|0|NONE|-|
|2|SHA1|20|
|3|SHA256|32|
|4|SHA512|64|
|5|SHA256_160 (actually SHA2 160 variant)|20|

The missing ID1 was MD5 whose support was dropped in apk-tools.

If this have a valid, non-`NONE` hash_alg, then the actual payload should (after the 2-byte header) be followed by a 16-byte ID, and the corresponding length of signature, defined in C as:

```c
struct adb_sign_v0 {
    struct adb_sign_hdr hdr;
    uint8_t id[16];
    uint8_t sig[];
};
```

In the example file there's no `SIG` block.

### The data block `ADB_BLOCK_DATA`

Such block must be after `ADB_BLOCK_ADB` and cannot be before `ADB_BLOCK_SIG`

The block begins with a 8-byte header, including a u32 `path_idx` field for the 1-started ID of corresponding `PATH` element, and a u32 `file_idx` field for the 1-started ID of corresponding `FILE` element in that `PATH`, defined in C as:

```c
struct adb_data_package {
    uint32_t path_idx;
    uint32_t file_idx;
};
```

The file data follows directly after the header. Remember that the payload contains the header and each block starts at 8-byte boundary and aligns up to 8-byte boundary.

E.g. in the example file, right after `ADB_BLOCK_ADB` at `body[8:5660]`, pad that to 8-byte boundary `5664`, and reads the 4-byte type_size `u32(body[5664:5668]) == 0x8000007f`, so type for it is `0x8000007f >> 30 == 2`, for a `ADB_BLOCK_DATA`, and size for the payload including header is `0x7f`, 127, therefore the whole payload including header is `body[5664:+127] = body[5664:5791]`. In it, the path_idx is `u32(body[5668:5672]) == 3`, and file_idx is `u32(body[5672:5676]) == 1`, and actual data length is `127 (whole block) - 4 (block header) - 8 (data header) = 115`, and we can confirm it's `body[5676:5791] = body[5676:+115]`.

The file content reads as below:

```conf
config crowdsec 'crowdsec'
    option data_dir '/srv/crowdsec/data'
    option db_path '/srv/crowdsec/data/crowdsec.db'

```

And if we go back to look at the `PATH` block, we would know this is for folder `etc/config` and file `crowdsec`, perm `0o600` owned by `root:root`, with SHA256 checksum.
