---
layout: post
title:  "Merge multiple files by bytes on Windows"
date:   2024-05-08 09:10:00 +0800
categories: Windows
---

Merging multiple files on Linux is very easy as to run a simple `cat` command and redirect its output to the merged file.

Merging multiple files on Windows however, needs more complex operation

Assuming we would want to do the same as the following command on Linux:
```
cat * > merged
```
which is expanded to the following command by shell:
```
cat part1 part2 part3 > merged
```

We have the following ways to achieve this on Windows (not counting Cygwin / MingW64, as that's pretty much cheating)

## Using cmd
The `copy` command is `cmd`'s **built-in** instead of a system tool, to merge multiple files, use a command like:
```
copy /B part1 + part2 + part3 merged
```
in which:
- `/B` means copying bytewise, this is needed unless you want to merge them text-wise
- The merged output must be the last `argument`, the `copy` built-in does not write to stdout
- Each to-be-merged components shall be connected with `+` as seperator between their arguments, as `+` itself is an argument
- `cmd` does not expand globs, every part much be typed as arguments explicitly (even if it expands, that won't be of much use anyway due to the `+` argument needed in between)

Note that above command could also be written as:
```
copy /B part1+part2+part3 merged
```
But I'd recommend against this pattern as it makes weak seperation between files to be mergd.

## Using PowerShell < 6.0
The `Get-Content` is `PowerShell`'s **built-in**, to merge multiple files, use a command like:
```
Get-Content -Encoding Byte -ReadCount 0x100000 part* | Set-Content -Encoding Byte merged
```
in which:
- `-Encoding` declares how to decode/encode files, `Byte` here means do not decode/encode bytes in file and keep them as-is
- `-ReadCount` declares how many words to read at once, for `Byte` encoding this means how many bytes to read at once, the default is `1` and that's insane for byte-reading. I've set it to `0x100000` which means 1MiB for each read

Note that the above command is very slow, slower than `cmd`, as bytes need to be parsed into native objects into PowerShell, this runs a few MiBs above 10 MiB/s, be patient.

## Using PowerShell >= 6.0
PowerShell 6.0 brings `-AsByteStream` argument to `Get-Content` built-in, so the above command could be rewritten as:
```
Get-Content -AsByteStream -ReadCount 0x100000 part* | Set-Content -AsByteStream merged
```
This is more efficient than the < 6.0 version and should run faster, but still much much slower than `cmd`

Note that PowerShell >= 6.0 also drops `-Encoding Byte` for `Get-Content` and `Set-Content`, it's not possible to compare the performance between these versions.