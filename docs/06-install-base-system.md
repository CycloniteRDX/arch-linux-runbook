# 06 — Install the base system

## Goal

Install a small but operational Arch Linux system into `/mnt`, including every
package required by the canonical LUKS2, LVM, UKI, systemd-boot, Secure Boot,
and NetworkManager path.

This chapter does not configure the installed system or make it bootable.

## Prerequisites

- Chapter 05 has been completed successfully.
- Root, home, and the EFI System Partition are mounted below `/mnt`.
- `/dev/vg0/swap` is active.
- The live environment has working internet access and a credible system time.

## Recheck the installation target

```bash
findmnt /mnt
findmnt /mnt/home
findmnt /mnt/boot
swapon --show
```

Confirm that the sources and targets match chapter 05. In particular,
`/dev/nvme0n1p1` must be mounted at `/mnt/boot` with `fmask=0177` and
`dmask=0077` before installing the kernel.

## Check network, time, and mirrors

```bash
ping -c 3 archlinux.org
timedatectl
grep '^Server' /etc/pacman.d/mirrorlist
```

Continue only if DNS works, the clock is credible, and the mirror list contains
enabled `Server` entries. The installation image normally provides a usable
mirror list, so it does not need to be replaced routinely.

### Optional: replace a slow mirror list

Run this only if the existing mirrors are unavailable or consistently slow:

```bash
reflector --country Spain,Portugal,France,Germany --age 24 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Then inspect the enabled entries again:

```bash
grep '^Server' /etc/pacman.d/mirrorlist
```

This changes the mirror list used by `pacstrap`; it does not install Reflector
as a persistent service in the target system.

## Package set

| Package | Purpose |
| --- | --- |
| `base` | Minimal Arch userspace, pacman, and systemd |
| `linux` | Standard Arch kernel |
| `linux-firmware` | Firmware bundle for laptop hardware |
| `amd-ucode` | AMD processor microcode |
| `mkinitcpio` | Initramfs and UKI generation |
| `systemd-ukify` | UKI assembly used by mkinitcpio |
| `cryptsetup` | LUKS2 userspace and initramfs support |
| `lvm2` | LVM userspace and initramfs support |
| `e2fsprogs` | ext4 checking and maintenance tools |
| `dosfstools` | FAT checking and maintenance tools |
| `efibootmgr` | UEFI NVRAM boot-entry management |
| `sbctl` | Secure Boot key and signature management |
| `networkmanager` | Persistent wired and wireless networking |
| `sudo` | Privilege delegation for the regular user |
| `micro` | Straightforward console text editor |
| `man-db`, `man-pages` | Local command and system documentation |

The `base` package already depends on systemd, and systemd-boot is supplied by
the systemd package. NetworkManager currently pulls in `wpa_supplicant` as its
default Wi-Fi backend, so neither `dhcpcd` nor `iwd` is required here.

Package versions are intentionally not pinned. One `pacstrap` transaction uses
the mutually compatible versions currently published by the enabled official
repositories.

## Install the packages

```bash
pacstrap -K /mnt base linux linux-firmware amd-ucode mkinitcpio systemd-ukify cryptsetup lvm2 e2fsprogs dosfstools efibootmgr sbctl networkmanager sudo micro man-db man-pages
```

`-K` initializes a new pacman keyring inside the target. `pacstrap` also seeds
the target with the live environment's current mirror list.

Do not disable package-signature checking or add `--noconfirm` merely to bypass
an unexpected prompt or error. Resolve clock, mirror, keyring, and package
conflicts before continuing.

## Initial kernel artifacts

Installing the kernel triggers mkinitcpio before the final encrypted-root and
UKI configuration exists. `/mnt/boot` may therefore contain a traditional
kernel and initramfs after `pacstrap` finishes.

This is a temporary and expected state. Later chapters replace the kernel
preset with direct UKI output and regenerate the artifacts. Do not delete or
sign individual files yet.

Broad fallback images can also report missing firmware for hardware that is
not present in these ThinkPads. A warning is not automatically a failure, but
any error that aborts mkinitcpio or the package transaction must be resolved.

## Verify the installation

```bash
arch-chroot /mnt pacman -Q linux linux-firmware amd-ucode mkinitcpio systemd-ukify cryptsetup lvm2 efibootmgr sbctl networkmanager
ls /mnt/boot
findmnt /mnt/boot
```

Confirm that:

- Every queried package is installed.
- The kernel installation created files below the mounted ESP.
- `/mnt/boot` still refers to `/dev/nvme0n1p1`.
- The ESP remains mounted read-write with the restrictive masks.
- `pacstrap` and its triggered mkinitcpio run completed without fatal errors.

The target still lacks `fstab`, locale, hostname, users, enabled networking,
encrypted-root initramfs configuration, UKIs, boot-manager installation, and
Secure Boot keys.

## Deliberately deferred packages

`base-devel`, kernel headers, Git, OpenSSH, firewalls, zram, graphics, audio,
Bluetooth, printing, power management, and graphical applications are not
required to reach the first working TTY. They belong to the post-install
repository or to a later optional runbook appendix.

## Sources

- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: pacstrap(8)](https://man.archlinux.org/man/pacstrap.8)
- [Arch Linux: base package](https://archlinux.org/packages/core/any/base/)
- [Arch Linux: NetworkManager package](https://archlinux.org/packages/extra/x86_64/networkmanager/)
- [Arch manual: reflector(1)](https://man.archlinux.org/man/reflector.1)

## Next step

Continue with chapter 07 to generate `/etc/fstab` once, inspect it, and verify
the UUIDs, mount points, filesystem types, and ESP permission masks.
