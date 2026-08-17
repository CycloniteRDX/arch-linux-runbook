# 00 — Installation profile

## Goal

Define the exact system that this runbook will build. Later chapters use these
choices as fixed assumptions instead of presenting alternatives during the
installation.

This chapter does not modify the computer.

## Supported target

| Item | Canonical value |
| --- | --- |
| Computer | Lenovo ThinkPad T14 Gen 1 AMD |
| Tested processors | AMD Ryzen 5 PRO 4650U and Ryzen 7 PRO 4750U |
| Memory | 16 GiB or more |
| Internal drive | One approximately 512 GB NVMe SSD |
| Firmware boot mode | UEFI only |
| Installation medium | Current official Arch Linux ISO |
| Architecture | x86-64 |

The procedure may work on other UEFI computers, but their device names,
firmware menus, drivers, and Secure Boot behavior must be reviewed separately.

## Values that must be confirmed

The examples use the following names:

| Purpose | Runbook value |
| --- | --- |
| Target disk | `/dev/nvme0n1` |
| EFI System Partition | `/dev/nvme0n1p1` |
| LUKS2 partition | `/dev/nvme0n1p2` |
| Open LUKS mapping | `cryptlvm` |
| LVM volume group | `vg0` |
| Root logical volume | `root` |
| Home logical volume | `home` |
| Swap logical volume | `swap` |
| Regular user | `neon` |
| Hostname | A unique lowercase name chosen for each laptop |

`/dev/nvme0n1` must never be assumed from memory. It will be verified from the
live ISO before any partitioning command is executed.

The names `cryptlvm` and `vg0` are labels chosen by this project. They do not
need to match the names used by an existing Arch installation.

## Storage layout

The reference 512 GB SSD reports approximately 476.9 GiB to Linux.

| Region | Approximate size | Purpose |
| --- | ---: | --- |
| GPT metadata | Automatic | Partition table |
| Partition 1 | 1 GiB | FAT32 EFI System Partition |
| Partition 2 | 400 GiB | LUKS2 encrypted container |
| Unpartitioned tail | About 75.9 GiB | Intentional host-provided SSD spare area |

The unpartitioned tail is deliberate. It gives the SSD controller additional
unused flash-address space when those blocks are known to be unused. It may
help garbage collection and sustained-write behavior, but it is not a backup,
a security boundary, or a substitute for TRIM and the SSD's factory spare
area.

Inside the opened LUKS2 container:

| Logical volume | Size | Format | Mount point |
| --- | ---: | --- | --- |
| `vg0/swap` | 16 GiB | Linux swap | None |
| `vg0/root` | 192 GiB | ext4 | `/` |
| `vg0/home` | Remaining usable space | ext4 | `/home` |
| Volume-group slack | About 256 MiB | Unallocated extents | None |

The small amount left free in `vg0` prevents the layout from consuming every
last physical extent. It is only a buffer; it is not large enough to be a
useful snapshot reserve.

## Filesystem policy

- `/` uses ext4 with 1% reserved blocks.
- `/home` uses ext4 with 0% reserved blocks.
- `/boot` uses FAT32 because it is the EFI System Partition.
- `/boot` is mounted with `fmask=0077,dmask=0077` so only root can traverse or
  read its contents through the mounted filesystem.
- Continuous ext4 `discard` mount options are not used.
- Periodic TRIM and LUKS discard propagation are configured after the first
  boot in `arch-linux-post-install`.

The root reserve gives privileged processes some recovery space if `/` becomes
nearly full. Reserving that space on `/home` provides little benefit, so its
reserved-block percentage is zero.

## Encryption and unlock policy

- The LUKS2 container covers swap, `/`, and `/home`.
- The EFI System Partition remains unencrypted because the firmware must read
  its EFI executables.
- The signed UKI protects the integrity of the kernel, initramfs, microcode,
  and embedded kernel command line stored on that unencrypted partition.
- Unlocking uses a strong LUKS passphrase entered during boot.
- Automatic TPM2 unlocking is outside the canonical installation.
- Recovery depends on the passphrase, not on the TPM or a network service.

## Swap, zram, and suspend

- A 16 GiB disk-backed swap logical volume is created during installation.
- Hibernation and `resume=` configuration are not included.
- Normal suspend remains supported.
- Zram is added after the first boot with a higher swap priority than the disk
  swap.
- The disk swap remains available as a low-priority fallback.

The swap size therefore does not promise hibernation support.

## Boot and trust chain

The canonical path is:

1. UEFI firmware.
2. Signed `systemd-boot` EFI executable.
3. Signed Arch Linux unified kernel image.
4. `systemd`-based initramfs.
5. LUKS2 passphrase prompt.
6. LVM activation and ext4 root mount.
7. systemd boot to a TTY.

The normal and fallback UKIs are generated directly during installation. An
unsigned, non-UKI boot path is not part of the finished system.

`systemd-boot` is not technically required to execute a UKI. It is retained
because it provides a simple boot menu, firmware integration, fallback entry,
and predictable update workflow.

Secure Boot keys are managed with `sbctl`. Microsoft vendor certificates are
retained when supported by the firmware so that signed firmware components and
external boot media are not unnecessarily excluded. Key enrollment is treated
as a hardware-sensitive operation and receives its own chapter.

## Base system choices

| Area | Choice |
| --- | --- |
| Kernel | `linux` |
| CPU microcode | `amd-ucode` |
| Initramfs | `mkinitcpio` with systemd hooks |
| UKI builder | `ukify` through the mkinitcpio preset |
| Boot manager | `systemd-boot` |
| Secure Boot manager | `sbctl` |
| Network manager | NetworkManager |
| Time zone | `Europe/Madrid` |
| System locale | `en_US.UTF-8` |
| Virtual-console keymap | `us` |
| Main filesystem | ext4 |

One regular user named `neon` is created with `wheel` membership and verified
`sudo` access. Each ThinkPad receives a different hostname.

## Completion criteria

The runbook is complete only when all of the following succeed:

- The firmware starts the installed boot manager in UEFI mode.
- Secure Boot is enabled and the active boot chain is signed.
- Both normal and fallback UKIs are listed by `systemd-boot`.
- The LUKS2 passphrase unlocks the encrypted system.
- Root and home mount from the expected LVM logical volumes.
- The regular user can log in at a TTY and use `sudo`.
- NetworkManager is enabled and networking can be established.
- No graphical environment is required for the verification.

## Out of scope

This runbook does not configure zram, periodic TRIM, firewall rules, broader
hardening, power tuning, desktop services, applications, Niri, or dotfiles.
Those steps begin in `arch-linux-post-install`.

## Sources

- [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [ArchWiki: LVM on LUKS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)
- [ArchWiki: Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [ArchWiki: Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [systemd-stub manual](https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html)
- [systemd-boot manual](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html)
- [sbctl upstream repository](https://github.com/Foxboron/sbctl)

## Next step

Continue with chapter 01 to boot the installation medium, verify UEFI mode,
establish networking, and identify the target disk safely.
