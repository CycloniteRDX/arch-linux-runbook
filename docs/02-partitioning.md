# 02 — Partitioning

## Goal

Replace the target SSD's existing partition table with the canonical GPT
layout:

| Partition | Size | GPT type | Later use |
| --- | ---: | --- | --- |
| `/dev/nvme0n1p1` | 1 GiB | EFI System | FAT32, mounted at `/boot` |
| `/dev/nvme0n1p2` | 400 GiB | Linux filesystem | LUKS2 container |
| Unpartitioned tail | About 75.9 GiB | None | Intentional SSD spare area |

This chapter creates partitions only. Encryption, LVM, filesystems, and mounts
are handled later.

## Prerequisites

- Chapter 01 has been completed successfully.
- Every required file from the target SSD has been backed up.
- The intended internal disk is `/dev/nvme0n1`.
- No external disk can be confused with the installation target.
- Existing operating systems and partitions do not need to be preserved.

## Final safety check

> [!CAUTION]
> The `w` command used later in `fdisk` replaces the partition table on
> `/dev/nvme0n1`. All existing partitions and operating systems on that SSD
> will become inaccessible. Recovery is not part of this runbook.

Verify the complete disk identity again:

```bash
lsblk -d -o NAME,PATH,SIZE,MODEL,SERIAL,TRAN
fdisk -l /dev/nvme0n1
```

Check that none of its current partitions is mounted and that no old swap is
active:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1
swapon --show
```

Stop if the model, serial number, size, or path is not the intended internal
SSD. Do not continue merely because the device happens to be named
`/dev/nvme0n1`.

## Open fdisk

Acquire an exclusive lock while editing the disk:

```bash
fdisk --lock=yes /dev/nvme0n1
```

All changes remain in memory until `w` is entered. Use `q` at any point before
`w` to exit without writing them.

Useful interactive commands for this procedure:

| Command | Action |
| --- | --- |
| `m` | Show help. |
| `p` | Print the proposed partition table. |
| `g` | Create a new empty GPT table in memory. |
| `n` | Create a partition. |
| `t` | Change a partition type. |
| `l` | List known partition types. |
| `d` | Delete one partition from the in-memory table. |
| `q` | Quit without saving. |
| `w` | Write the table and exit. |

Interactive commands are entered without a leading hyphen. Creating a new GPT
table with `g` makes deleting every old partition individually unnecessary.

## Create the GPT layout

### 1. Inspect the current table

At the `fdisk` prompt:

```text
p
```

Confirm once more that the displayed disk is `/dev/nvme0n1`.

### 2. Create a new GPT table

```text
g
```

This replaces the old layout in memory. It is not written yet.

### 3. Create the EFI System Partition

Enter:

```text
n
```

Accept the default partition number and first sector by pressing `Enter` at
both prompts. At the last-sector prompt, enter:

```text
+1G
```

Change the new partition's type:

```text
t
uefi
```

The `uefi` alias selects the EFI System partition type. No separate bootable
flag is required on GPT.

### 4. Create the encrypted-system partition

Enter:

```text
n
```

Again, accept the default partition number and first sector. At the last-sector
prompt, enter:

```text
+400G
```

Leave this partition at fdisk's default `Linux filesystem` type. The next
chapter writes the actual LUKS2 header and `crypto_LUKS` signature. The GPT type
is descriptive metadata and does not provide encryption.

If `fdisk` reports old filesystem signatures inside the new partition ranges,
confirm their removal only after rechecking that `/dev/nvme0n1` is the intended
disk.

### 5. Review before writing

```text
p
```

The proposed table must contain exactly:

- A 1 GiB EFI System partition.
- A 400 GiB Linux filesystem partition.
- No third partition.

If any value is wrong, use `q` and begin again. Do not try to rescue an unclear
in-memory layout merely to avoid repeating these few steps.

### 6. Write the partition table

> [!CAUTION]
> The next command commits the destructive change.

```text
w
```

`fdisk` writes the GPT, asks the kernel to reread it, synchronizes the change,
and exits.

## Verify the result

```bash
fdisk -l /dev/nvme0n1
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1
sfdisk --list-free /dev/nvme0n1
```

Expected result:

- `/dev/nvme0n1p1` is approximately 1 GiB and type `EFI System`.
- `/dev/nvme0n1p2` is approximately 400 GiB and type `Linux filesystem`.
- Approximately 75.9 GiB remains unpartitioned at the end of the SSD.
- Neither partition has been formatted or mounted by this chapter.

Small free regions used for GPT alignment and metadata are normal. The large
free region at the end is the intentional spare area.

## Sources

- [ArchWiki: Partitioning](https://wiki.archlinux.org/title/Partitioning)
- [Arch manual: fdisk(8)](https://man.archlinux.org/man/fdisk.8)
- [Arch manual: lsblk(8)](https://man.archlinux.org/man/lsblk.8)
- [Arch manual: sfdisk(8)](https://man.archlinux.org/man/sfdisk.8)

## Next step

Continue with chapter 03 to create and open the LUKS2 container on
`/dev/nvme0n1p2`.
