# 03 — LUKS2 encryption

## Goal

Initialize `/dev/nvme0n1p2` as a LUKS2 container and open it as
`/dev/mapper/cryptlvm`.

The EFI System Partition at `/dev/nvme0n1p1` remains unencrypted. LVM and the
filesystems will be created inside the open mapping in the next chapter.

## Prerequisites

- Chapter 02 has been completed successfully.
- `/dev/nvme0n1p1` is the 1 GiB EFI System Partition.
- `/dev/nvme0n1p2` is the 400 GiB encrypted-system partition.
- Neither partition is mounted or otherwise in use.
- The active console keymap is the one that will be configured for early boot.
- A long, unique LUKS passphrase has been chosen.

Choose a passphrase that can be entered reliably with the selected keyboard
layout. Do not reuse a password from an online account. Characters are not
displayed while a passphrase is entered.

## Verify the target partition

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1
```

Continue only if `/dev/nvme0n1p2` is the intended 400 GiB partition and has no
mount point. Do not run `luksFormat` on `/dev/nvme0n1`,
`/dev/nvme0n1p1`, or an existing mapper device.

## Create the LUKS2 container

> [!CAUTION]
> The following command writes a new LUKS header to `/dev/nvme0n1p2`. An old
> LUKS header at that location will be replaced and its encrypted data will
> become inaccessible. This operation does not securely erase the partition's
> complete data area.

```bash
cryptsetup luksFormat --type luks2 --verify-passphrase /dev/nvme0n1p2
```

Read the displayed target path carefully. At the confirmation prompt, enter:

```text
YES
```

Then enter the new passphrase twice.

The command intentionally uses cryptsetup's current defaults for the cipher,
volume-key size, password-based key derivation, alignment, and encryption
sector size. Do not copy hardware-specific values from another computer.

## Inspect the new header

```bash
cryptsetup luksDump /dev/nvme0n1p2
```

Confirm that the output reports:

- `Version: 2`.
- One populated keyslot.
- No error reading the LUKS header.

The detailed cipher and PBKDF values may vary with the cryptsetup version and
the hardware used during formatting.

## Open the container

```bash
cryptsetup open /dev/nvme0n1p2 cryptlvm
```

Enter the same passphrase. This creates the temporary device-mapper path:

```text
/dev/mapper/cryptlvm
```

The mapping name `cryptlvm` is used consistently throughout the rest of this
runbook. This command does not enable discard passthrough; that policy belongs
to the post-install configuration.

## Verify the open mapping

```bash
cryptsetup status cryptlvm
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1
```

Expected result:

- `cryptsetup status` reports that `/dev/mapper/cryptlvm` is active.
- Its backing device is `/dev/nvme0n1p2`.
- `/dev/nvme0n1p2` is identified as `crypto_LUKS`.
- `cryptlvm` appears below that partition with type `crypt`.
- The open mapping has no filesystem and is not mounted yet.

Leave `cryptlvm` open. The next chapter uses it as the LVM physical volume.

## Recovery note

A LUKS header backup is valuable because damage to the header or all usable
keyslots can make the encrypted data irrecoverable. Create that backup after
installation and keep it on trusted external storage, not on this SSD or its
unencrypted EFI System Partition.

## Sources

- [ArchWiki: dm-crypt device encryption](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption)
- [Arch manual: cryptsetup-luksFormat(8)](https://man.archlinux.org/man/cryptsetup-luksFormat.8)
- [Arch manual: cryptsetup-open(8)](https://man.archlinux.org/man/cryptsetup-open.8)
- [Arch manual: cryptsetup(8)](https://man.archlinux.org/man/cryptsetup.8)

## Next step

Continue with chapter 04 to create the LVM physical volume, volume group,
logical volumes, swap area, and ext4 filesystems inside `cryptlvm`.
