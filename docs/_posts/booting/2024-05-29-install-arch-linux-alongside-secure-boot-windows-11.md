---
layout: post
title:  "Installing Arch Linux Alongside Secure Boot Windows 11"
date:   2024-05-29 12:00:00 +0800
categories: booting
---

Recently, I got myself a new Thinkbook 16+ as my new and only laptop, which came with Windows 11 pre-installed bundled with a copy of Microsoft Office 2021. The laptop came with Secure Boot enabled, and later full disk Bitlocker enabled after a successful Microsoft account login after I activated the Office copy.

As always I want to have Arch Linux on my new device, for the use cases I planned I would have it dual booting with Windows like my home desktop. For an Arch installation, instead of disabling the Secure Boot and/or Bitlocker as a whole, I figured it would be a better idea to keep Secure Boot and install Arch onto a LUKS-encrypted btrfs rootfs, disallowing both unauthorized boot and root access.

So the setup would be:
- Secure Boot is enabled
- The Platform Key (PK) is kept as is, i.e. the Microsoft's root public cert.
- Windows 11 lives in its 256 GiB Bitlocker encrypted NTFS root, key enrolled to TPM
- Arch Linux lives in a ~700 GiB LUKS2 encrypted Btrfs root, key not enrolled (on purpose)
- `/boot` on Arch Linux lives in the encrypted root, not ESP
- EFI files and configs of Windows 11 are kept as-is
- Fedora's signed shim is installed into ESP as `/efi/EFI/ArchLinux/shimx64.efi`
- Machine Owner Key (MOK) manager is installed alongside shim as `/efi/EFI/ArchLinux/mmx64.efi`, and MOK alongside it
- `/efi` is mounted automatically by systemd, a signed Unified Kernerl Image (UKI) is stored under ESP as `/efi/EFI/ArchLinux/grubx64.efi` (don't be tricked by name, it's UKI, not GRUB)

So the booting chain is either one of:
- UEFI
  - Fedora shim (`/efi/EFI/ArchLinux/shimx64.efi`) 
    - MOK manager (`/efi/EFI/ArchLinux/mmx64.efi`)
      - Manually add MOK `/efi/EFI/ArchLinux/MOK.cer` to key store
    - UKI signed with MOK (`/efi/EFI/ArchLinux/grubx64.efi`)
      - Booster finds rootfs
        - Manually unlook the rootfs with password (key not enrolled on purpose)
          - Arch Linux
  - Windows Bootloader
    - Bitlocker unlocked with enrolled key to TPM
      - Windows 11

Do note some points that I set up on purpose:
- There's no bootloader for Linux, shim loads a UKI directly that pretends to be GRUB. This is because fedora's shim is hard-coded to always chain-load GRUB.
- The rootfs of Arch Linux is an encrypted LUKS volume, unlocked with a password, without a key enrolled to either TPM or Fido key, as I don't want risk the rootfs being possible to be unlocked without the actual existence of myself and my own allowance (enrolling to TPM gives away both, enrolling to Fido gives away the prior).

## Important Installation steps
Steps that're already documented in the [Installation Guide on Arch Linux](https://wiki.archlinux.org/title/Installation_guide) would be omitted, I'll only mark the important steps that would go differently.

### Shrink Windows partition
Do this in Windows, not from Linux. Also remember to create a new partition to take the new free space. It's not needed to create a filesystem on the new partition, though.

### Windows recovery key backup
Back up your Windows BitLocker recovery key on [Microsoft accounts page](https://account.microsoft.com/devices/recoverykey), temporarily, this would be used later after we tempered with TPM and the enrolled BitLocker key considered tainted by Windows boot loader.

### UEFI BIOS adjustment
Disable secure boot, because to boot Arch ISO under Secure Boot we'll either need to use signed blob beforehand and add the key of kernel right now, tainting SB and TPM, which is not what we really want and needs to be removed later, or just disable secure boot. Don't worry, secure boot would be re-enabled after we installed Arch Linux.

Allow Microsoft 3rd Party UEFI CA certificates to be installed, this option is usually alongside Secure Boot. This is needed for the Fedora signed shim to work.

### LUKS on a partition
Follow the installtion guide until _1.8 Update the system clock_, now jump to [the steps on Arch Linux](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition) to create the rootfs, only follow it till _2.2 Preparing non-boot partitions_, then jump back to the installation guide and follow _1.11 Mount the file systems_ onwards.

This should ensure a rootfs on LUKS on a partition layout, i.e. you should have your whole rootfs encrypted, even the swap (as swapfile). Note that you should not have a seperate boot partition, nor should you use the existing ESP as `/boot`, instead, `/boot` should be right under the rootfs.

### Initcpio generator
When installing packages, I'll recommend to choose `booster` as initcpio generator over `mkinitcpio`, which (`mkinitcpio`) is the default. So the example pacstrap line would looks like this (with an extra `booster`):
```
pacstrap -K /mnt base linux-zen linux-firmware booster
```
An explicit `booster` is needed as all kernel packages need `initramfs` virtual pacakge, and `mkinitcpio` is the default choice.

I recommend `booster` because it's fast and provides an enough feature set that we need for this scenario (note `mkinitcpio` supports automatical UKI generating and signing, but it expects key in world-readable place, and hardcodes some paths). It only packs two statically linked go binaries, one as a generator and another as an init implementation, so both initcpio generating and booting go very fast.

### Enter chroot and do basic setups
Follow section _3 Configure the system_ in Installation Guide, skip `3.6 Initramfs` and `3.8 Boot loader`. Don't exit from the chroot yet, we have a few other things to do:
- Install a TUI-based text editor, `vim` or `nano`, if `vim` then I recommend to `ln -s vim /usr/bin/vi`
- Install `sudo`, then `visudo` to allow the `wheel` group to use `sudo` without password (Uncomment line `%wheel ALL=(ALL:ALL) NOPASSWD: ALL`)
- Install your favorite shell, mine's `fish`
- Add your own user with appened `wheel` group (e.g. `useradd --groups wheel --shell /usr/bin/fish --create-home [USERNAME]`)
- Set up password for your own user, don't switch to it yet
- Install `systemd-ukify` (to generate UKI), `sbsigntools` (for signing UKI), `shim-signed` (for signed EFI), the last one comes from AUR and could be installed from `archlinuxcn` repo (if you want, add and enable the repo following [Arch Linux CN Wiki](https://wiki.archlinuxcn.org/wiki/Arch_Linux_%E4%B8%AD%E6%96%87%E7%A4%BE%E5%8C%BA%E4%BB%93%E5%BA%93))

### Key generating
We'll need a key to function as the Machine Owner Key (MOK), with its private key owned by ourselves and public key enrolled to TPM. 

As this key is associated with you, the machine owner and the actual user, I recommend to generate the key as the non-root user identity (`su - [USERNAME]` first) and store it privately.

The generating goes like folllowing:
```
umask 077
mkdir -p ~/.pki/secure_boot
cd ~/.pki/secure_boot
openssl req -newkey rsa:2048 -nodes -keyout MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Machine Owner Key/" -out MOK.crt
openssl x509 -outform DER -in MOK.crt -out MOK.cer
```

You would have the following files under `~/.pki/secure_boot`:
- MOK.key: the private key of the key pair, it's used to sign UKIs later
- MOK.crt: the certificate, containing the public key of the value, you can dump its info by `openssl x509 -in MOK.crt -noout -text`
- MOK.cer: the certificate, similar to `.crt` but stored in a format that's used by UEFI Secure Boot

Of which, you **must keep the private key safe**. On the other hand, leaking of certificate (public key) is not destructive, but loss of it still is.

You should store the `MOK.cer` file under ESP:

```
sudo install --owner root --group root --mode 400 --no-target-directory -D MOK.cer /efi/EFI/ArchLinux/MOK.cer
```

You can delete `MOK.cer` from `~/.pki/secure_boot` after this, it could be regenerated as long as you have `MOK.crt`

### UKI generating and signing
Do this section still as the non-root user

Prepare a UKI generating config like following:
```
kernel="${kernel:-linux-zen}"
echo "[UKI]
Linux=/boot/vmlinuz-"${kernel}"
Initrd=/boot/amd-ucode.img /boot/booster-"${kernel}".img
Cmdline=rd.luks.uuid=$UUID=root root=/dev/mapper/root rootflags=subvol=@,compress=3 rw
" | sudo install -DTm 644 /dev/stdin /etc/ukify.d/"${kernel}".conf
```
Adapt this to your need, notably **you should replace `$UUID` with the actual UUID of your LUKS volume (not the UUID of the fs on LUKS)**, and you should drop or update `rootflags` depending on your rootfs.

Generate the UKI and sign it then install it like following:
```
mkdir -p ~/.cache/secure-boot
kernel="${kernel:-linux-zen}"
ukify build --config /etc/ukify.d/"${kernel}".conf --output ~/.cache/secure-boot/uki-"${kernel}".efi
sbsign --key ~/.pki/secure-boot/MOK.key --cert ~/.pki/secure-boot/MOK.crt ~/.cache/secure-boot/uki-"${kernel}".efi 
sudo install ~/.cache/secure-boot/uki-"${kernel}".efi.signed /efi/EFI/ArchLinux/grubx64.efi
```

You should store the above as a script for the automatical generation we would deploy later (e.g. as `~/Scripts/uki-update.sh`)

### Shim and MOK manager installing
Install the shim loader and MOK manager
```
sudo pacman -Syu shim-signed efibootmgr
sudo cp /usr/share/shim-signed/{shim,mm}x64.efi /efi/EFI/ArchLinux/
```
Create a UEFI boot loader entry for the shim loader
```
sudo efibootmgr --create --disk [root drive, e.g. /dev/nvme0n1] --part [ESP no, e.g. 1] --label 'Arch Linux (shim loader, zen kernel, direct load)' --loader '\EFI\ArchLinux\shimx64.efi' --unicode
```

### Enable Secure Boot and add Machine Owner Key (MOK)
Reboot and enable Secure Boot and boot into our new UEIF boot entry.

The shim loader would fail to verify `grubx64.efi` as its key is not enrolled yet. Select to enter MOK manager, then navigate to find your MOK file (`EFI/ArchLinux/MOK.cer`) and import it.

After importing, reboot again, this should boot you into the system.

After this, you can safely delete the MOK manager and MOK file from ESP to reduce the exposure of your MOK and prevent others from importing their keys as MOK.

If you could boot successfully, continue to set up some systemd units to automatically update your signed UKI.

### Set up systemd units to automatically update the signed UKI
Make sure your generation script works when running as a normal user (`sh ~/Scripts/uki-update.sh`), then let's modify it to ensure it works with dynamic kernel name:

In `~/Scripts/uki-update.sh`, change the line that sets kernel to read arg1:
```
kernel="${1:-linux}"
```
Note that we didn't implement multi-kernel support, if you want to do that, you'll need either multiple folders under `/efi/EFI`, or use a bootloader, in whichever case you should write your own script

Then add the following service unit under `/etc/systemd/system`:

_uki-update@.service_
```
[Unit]
Description=Update and sign UKI for %i kernel
After=local-fs.target
Wants=local-fs.target # It would hint efi.mount, but not guaranteed
ConditionPathIsMountPoint=/efi # In case efi.mount is not generated
ConditionPathIsDirectory=/efi/EFI/ArchLinux
ConditionPathIsReadWrite=/efi/EFI/ArchLinux

[Service]
User=[USER NAME]
ExecStart=/bin/bash [USER HOME]/Scripts/uki-update.sh %i
```

Start the service unit and check if it works
```
sudo systemctl start uki-update@linux-zen.service
systemctl status uki-update@linux-zen.service
```

Then add the following path unit under `/etc/systemd/system`:

_uki-update@.path_
```
[Unit]
Description=Update and sign UKI for %i kernel once initcpio updated

[Path]
PathChanged=/boot/booster-%i.img

[Install]
WantedBy=paths.target
```

Start the path unit
```
sudo systemctl start uki-update@linux-zen.path
```

On a terminal, watch for the real-time log of the updater:
```
journalctl -u uki-update@linux-zen.service -f
```
On another terminal, regenerate the initcpio to check if the updater works:
```
sudo /usr/lib/booster/regenerate_images 
```
If you have new logs from updater, you can enable the path unit and let systemd update and sign the UKI automatically for you:
```
sudo systemctl enable uki-update@linux-zen.path
```

### Windows BitLocker key recovery

Since we've tempered with TPM, Windows bootloader would consider the current key store insecure to use, although the BitLocker key is still in there. Boot into Windows and bootloader would hint on BitLocker key recovery. Type in the recovery key we recorded earlier once and it would be enrolled again. The bootlodaer wouldn't request the recovery key on the next boots.