# 07 — Generate fstab

## Goal

Generate the installed system's `/etc/fstab` from the filesystems mounted below
`/mnt` and the active swap LV.

The final file must contain four active records:

| Source | Target | Type | Dump | Check pass |
| --- | --- | --- | ---: | ---: |
| Root filesystem UUID | `/` | `ext4` | 0 | 1 |
| Home filesystem UUID | `/home` | `ext4` | 0 | 2 |
| ESP filesystem UUID | `/boot` | `vfat` | 0 | 2 |
| Swap UUID | `none` | `swap` | 0 | 0 |

Actual UUIDs are generated independently on every installation and must never
be copied from an example.

## Prerequisites

- Chapter 06 has been completed successfully.
- Root, home, and the ESP remain mounted below `/mnt`.
- `/dev/vg0/swap` remains active.
- No additional filesystem is mounted below `/mnt`.

## Recheck what genfstab will describe

```bash
findmnt -R /mnt
swapon --show
```

`genfstab` records the current hierarchy, not the hierarchy the operator meant
to create. Confirm that:

- `/dev/vg0/root` is mounted at `/mnt`.
- `/dev/vg0/home` is mounted at `/mnt/home`.
- `/dev/nvme0n1p1` is mounted at `/mnt/boot`.
- The ESP options contain `fmask=0177,dmask=0077`.
- `/dev/vg0/swap` is the intended active swap.

Correct the mount tree before continuing if any relationship is wrong.

## Inspect the existing file

```bash
cat /mnt/etc/fstab
```

The `filesystem` package creates this file during `pacstrap`. On a fresh
installation it should contain no active filesystem or swap records. Stop if
it already contains an unexplained configuration.

## Understand `>` and `>>`

The command commonly shown in installation instructions uses `>>`:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

`>>` appends output to the file. It preserves existing content, but accidentally
running it twice also appends every generated record twice.

This runbook instead uses `>`:

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

`>` replaces the destination. Repeating the command with the same mount tree
does not accumulate duplicate records, but it also removes anything previously
stored in the file. It is appropriate here because this is a fresh installation
and the existing file has just been inspected. Do not use the same replacement
command casually on an established system.

## Generate the file once

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

`genfstab` detects the mounts below `/mnt`; `-U` writes filesystem UUIDs instead
of discovery-order device paths.

If the command reports an error, stop. Do not enter the chroot or reboot with
an empty or incomplete file.

## Read the generated file

```bash
cat /mnt/etc/fstab
```

Ignoring comments and blank lines, visually confirm exactly four active
records:

1. Root at `/`, type `ext4`, with final field `1`.
2. Home at `/home`, type `ext4`, with final field `2`.
3. The ESP at `/boot`, type `vfat`, with final field `2`.
4. Swap targeting `none`, type `swap`, with final field `0`.

Every source must begin with `UUID=` and every fifth field must be `0`. The ESP
options must include both `fmask=0177` and `dmask=0077`; additional FAT options
such as the codepage, character set, and error policy are normal.

The UUIDs in this file identify filesystems and swap. Do not substitute the
LUKS UUID or a GPT partition UUID.

If a small deliberate correction is needed, edit the installed file with:

```bash
arch-chroot /mnt micro /etc/fstab
```

Compare any questionable UUID with this output rather than guessing:

```bash
lsblk -f /dev/nvme0n1
```

## Validate fstab

Use the installed system's paths and userspace:

```bash
arch-chroot /mnt findmnt --verify --verbose
```

The command must finish with zero parse errors and zero errors. Read warnings
before deciding whether they are harmless.

This validation checks syntax and whether sources and targets can be resolved.
It does not replace the visual check that all four records refer to the intended
devices.

Do not run `mount -a` from the live environment: the target filesystems are
already mounted, and the live environment has a different root context.
`systemctl daemon-reload` is also unnecessary because the installed systemd
instance has not started yet.

## Mount-option policy

Do not add `discard` to the ext4 records. Periodic TRIM and dm-crypt discard
forwarding are separate post-install decisions. Do not add `nofail` to root,
home, or the ESP; all three are required components of this installation.

If generation must be repeated during this fresh installation, first correct
the mount or swap state, rerun the same `>` command, then repeat `cat` and the
`findmnt` validation.

## Sources

- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: genfstab(8)](https://man.archlinux.org/man/genfstab.8)
- [Arch manual: fstab(5)](https://man.archlinux.org/man/fstab.5)
- [Arch manual: findmnt(8)](https://man.archlinux.org/man/findmnt.8)

## Next step

Continue with chapter 08 to enter the installed system and configure its time
zone, locale, console keyboard, hostname, and root authentication.
