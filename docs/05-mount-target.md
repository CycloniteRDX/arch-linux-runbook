# 05 — Mount the installation target

## Goal

Assemble the future system below `/mnt` and activate its disk-backed swap:

| Source | Target |
| --- | --- |
| `/dev/vg0/root` | `/mnt` |
| `/dev/vg0/home` | `/mnt/home` |
| `/dev/nvme0n1p1` | `/mnt/boot` |
| `/dev/vg0/swap` | Active swap |

The resulting hierarchy is used by `pacstrap` and later becomes the installed
system's `/`, `/home`, and `/boot` layout.

## Prerequisites

- Chapter 04 has been completed successfully.
- `cryptlvm` and `vg0` are active.
- Root and home are ext4 filesystems.
- The EFI System Partition is FAT32.
- The swap LV contains a valid swap signature.
- None of these filesystems is already mounted.
- `/dev/vg0/swap` is not already active.

## Verify the filesystems

```bash
lsblk -f /dev/nvme0n1
swapon --show
```

Recheck the filesystem types and device paths from chapter 04. Stop if any
target already has a mount point or if `/dev/vg0/swap` is already listed.

## Mount root first

```bash
mount /dev/vg0/root /mnt
```

Root must be mounted before creating or using its child mount points. Anything
created below `/mnt` before this command would belong to the live environment
and would be hidden when root is mounted.

## Mount home

```bash
mount --mkdir /dev/vg0/home /mnt/home
```

The `--mkdir` option creates `/mnt/home` if it does not already exist.

## Mount the EFI System Partition

```bash
mount --mkdir -o fmask=0177,dmask=0077 /dev/nvme0n1p1 /mnt/boot
```

This installation mounts the ESP directly at `/boot`, not at `/boot/efi`.
`systemd-boot` and the unified kernel images will therefore reside on the same
FAT32 filesystem that the firmware can read.

FAT does not store normal Unix permission bits. The mount masks present files
as mode `0600` and directories as mode `0700`, limiting access from Linux to
root. They do not prevent the UEFI firmware from reading the partition. The
later `fstab` chapter verifies that these masks are preserved.

## Activate swap

```bash
swapon /dev/vg0/swap
```

This activates the encrypted 16 GiB swap LV for the remainder of the
installation. It does not configure hibernation.

## Verify the complete target

```bash
findmnt /mnt
findmnt /mnt/home
findmnt /mnt/boot
swapon --show
lsblk -f /dev/nvme0n1
```

Confirm that:

- `/dev/vg0/root` is mounted at `/mnt`.
- `/dev/vg0/home` is mounted at `/mnt/home`.
- `/dev/nvme0n1p1` is mounted at `/mnt/boot`.
- The `/mnt/boot` options include `fmask=0177` and `dmask=0077`.
- `/dev/vg0/swap` is active and approximately 16 GiB.
- All three filesystems are mounted read-write.

Leave every mount and the swap LV active. The next chapter installs the base
system into this hierarchy.

## Sources

- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: mount(8)](https://man.archlinux.org/man/mount.8)
- [Arch manual: swapon(8)](https://man.archlinux.org/man/swapon.8)

## Next step

Continue with chapter 06 to review mirrors and install the base system, kernel,
firmware, microcode, networking, and essential administration tools.
