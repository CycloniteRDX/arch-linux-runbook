# 04 — LVM and filesystems

## Goal

Create the canonical LVM layout inside `/dev/mapper/cryptlvm`, then format the
EFI System Partition, swap LV, root LV, and home LV.

| Device | Size | Format | Purpose |
| --- | ---: | --- | --- |
| `/dev/nvme0n1p1` | 1 GiB | FAT32 | EFI System Partition |
| `/dev/vg0/swap` | 16 GiB | Linux swap | Encrypted disk-backed swap |
| `/dev/vg0/root` | 192 GiB | ext4 | Root filesystem |
| `/dev/vg0/home` | Remaining space | ext4 | Home filesystem |
| Free space in `vg0` | 256 MiB | Unallocated | Space required by `e2scrub` |

Small differences in the displayed home and VG sizes are normal because LUKS
and LVM store metadata and LVM allocates space in whole extents.

## Prerequisites

- Chapter 03 has been completed successfully.
- `/dev/mapper/cryptlvm` is open and backed by `/dev/nvme0n1p2`.
- `/dev/nvme0n1p1` is the unmounted 1 GiB EFI System Partition.
- The logical-volume names `swap`, `root`, and `home` are available.
- No existing volume group named `vg0` needs to be preserved.

## Verify the starting state

```bash
cryptsetup status cryptlvm
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1
vgs
```

Continue only if `cryptlvm` is the empty decrypted mapping created in chapter
03. If `vgs` already lists a volume group named `vg0`, stop and identify it
before continuing.

## Create the physical volume and volume group

> [!CAUTION]
> The following commands write LVM metadata to `/dev/mapper/cryptlvm`. Verify
> that this mapping belongs to `/dev/nvme0n1p2` before continuing.

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg0 /dev/mapper/cryptlvm
```

`vg0` is the canonical volume-group name used throughout this runbook.

## Create the logical volumes

Create swap and root with fixed sizes:

```bash
lvcreate --size 16G --name swap vg0
lvcreate --size 192G --name root vg0
```

Temporarily reserve 256 MiB, then allocate all other free extents to home:

```bash
lvcreate --size 256M --name e2scrub-reserve vg0
lvcreate --extents 100%FREE --name home vg0
```

The temporary LV contains no filesystem or data. Remove only that LV to return
its 256 MiB to the volume group's free-space pool:

```bash
lvremove /dev/vg0/e2scrub-reserve
```

At the confirmation prompt, verify the complete LV path and enter `y`.

This procedure avoids creating an oversized home LV and shrinking it later.
The ArchWiki recommends leaving at least 256 MiB free in a volume group that
contains ext4 filesystems so `e2scrub` can create a temporary snapshot.

## Verify the LVM layout

```bash
pvs
vgs
lvs
```

Confirm that:

- `/dev/mapper/cryptlvm` is the only physical volume in `vg0`.
- `vg0` contains exactly `swap`, `root`, and `home`.
- `swap` is 16 GiB and `root` is 192 GiB.
- `home` occupies almost all remaining space.
- `VFree` is 256 MiB.

Do not format anything if the names or sizes are different.

## Understand the three independent reservations

| Reservation | Storage layer | Purpose |
| --- | --- | --- |
| About 75.9 GiB outside the partitions | SSD and GPT | Intentional SSD spare area |
| 256 MiB free inside `vg0` | LVM | Temporary `e2scrub` snapshot |
| 1% of the root filesystem | ext4 | Emergency space reserved for root |

These regions are unrelated and must not be added together when interpreting
the available capacity.

## Format the filesystems

> [!CAUTION]
> The following commands replace existing filesystem metadata on their exact
> targets. Recheck every device path before pressing `Enter`.

Create FAT32 on the EFI System Partition:

```bash
mkfs.fat -F 32 /dev/nvme0n1p1
```

Create the swap area:

```bash
mkswap --lock=yes /dev/vg0/swap
```

Create the ext4 root filesystem with a 1% superuser reserve:

```bash
mkfs.ext4 -m 1 /dev/vg0/root
```

Create the ext4 home filesystem without a superuser-only reserve:

```bash
mkfs.ext4 -m 0 /dev/vg0/home
```

Ext4 normally reserves 5% of its blocks for the superuser. On the 192 GiB root
LV that would reserve about 9.6 GiB, so this profile reduces it to 1%, or about
1.9 GiB. The separate home filesystem uses `-m 0` because filling `/home`
cannot consume the emergency blocks reserved inside the root filesystem.

The 16 GiB swap LV is encrypted because it resides inside LUKS2. It is kept as
a disk-backed fallback for the future zram configuration; hibernation is not
configured by this runbook.

## Verify the result

```bash
lsblk -f /dev/nvme0n1
pvs
vgs
lvs
```

Expected filesystem types:

- `/dev/nvme0n1p1`: `vfat` with FAT32.
- `/dev/nvme0n1p2`: `crypto_LUKS`.
- `/dev/mapper/cryptlvm`: `LVM2_member`.
- `/dev/vg0/swap`: `swap`.
- `/dev/vg0/root`: `ext4`.
- `/dev/vg0/home`: `ext4`.

Root and home remain unmounted, and swap remains inactive. The next chapter
mounts the installation target and activates swap.

## Sources

- [ArchWiki: LVM](https://wiki.archlinux.org/title/LVM)
- [Arch manual: pvcreate(8)](https://man.archlinux.org/man/pvcreate.8)
- [Arch manual: vgcreate(8)](https://man.archlinux.org/man/vgcreate.8)
- [Arch manual: lvcreate(8)](https://man.archlinux.org/man/lvcreate.8)
- [Arch manual: mke2fs(8)](https://man.archlinux.org/man/mke2fs.8)
- [Arch manual: mkswap(8)](https://man.archlinux.org/man/mkswap.8)
- [Arch manual: mkfs.fat(8)](https://man.archlinux.org/man/mkfs.fat.8)

## Next step

Continue with chapter 05 to mount root at `/mnt`, home at `/mnt/home`, and the
EFI System Partition at `/mnt/boot`, then activate the installation swap.
