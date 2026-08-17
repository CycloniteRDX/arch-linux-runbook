# 12 — systemd-boot

## Goal

Install systemd-boot on the EFI System Partition, configure a short boot menu,
and verify that it discovers the normal and fallback UKIs automatically.

The UKIs created in chapter 11 are Boot Loader Specification Type #2 entries.
They do not need separate files below `/boot/loader/entries`.

## Prerequisites

- Chapter 11 has been completed successfully.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- `/boot` is the mounted EFI System Partition.
- Both UKIs exist below `/boot/EFI/Linux`.
- Secure Boot is still disabled in the firmware.

Recheck the boot partition and the UKIs:

```bash
findmnt /boot
ls -lh /boot/EFI/Linux
```

Stop if `/boot` is not backed by `/dev/nvme0n1p1` or either UKI is missing.

## Install systemd-boot

```bash
bootctl --esp-path=/boot install
```

This installs systemd-boot in its normal location and also places a copy at
the standard UEFI fallback path. It should create at least:

```text
/boot/EFI/systemd/systemd-bootx64.efi
/boot/EFI/BOOT/BOOTX64.EFI
/boot/loader/loader.conf
/boot/loader/random-seed
```

It also attempts to create a firmware entry named `Linux Boot Manager`. The
file installation may still succeed if firmware-variable registration is not
available from the chroot, so that entry is checked later in this chapter.

## Configure the boot menu

Open the systemd-boot configuration:

```bash
micro /boot/loader/loader.conf
```

Replace its contents with:

```text
default arch-linux.efi
timeout 3
console-mode max
editor no
```

Save and close Micro, then review the file:

```bash
cat /boot/loader/loader.conf
```

This configuration selects the normal UKI, displays the menu for three
seconds, requests the highest firmware console mode, and disables interactive
kernel-command-line editing.

Use the entry ID `arch-linux.efi`, not `arch.conf`. Do not create
`/boot/loader/entries/arch.conf`: systemd-boot discovers both Type #2 UKIs
directly from `/boot/EFI/Linux`.

`bootctl set-default` is useful for changing the default later, but it stores
the selection in an EFI variable that overrides `loader.conf`. The initial
installation therefore keeps the reproducible default in the file alone.

## Verify automatic UKI discovery

```bash
bootctl --esp-path=/boot is-installed
bootctl --esp-path=/boot list
```

The first command must report that systemd-boot is installed. The list must
contain both of these Type #2 entries:

```text
arch-linux.efi
arch-linux-fallback.efi
```

The normal image should be marked as the default. If an `arch.conf` entry
appears, remove that redundant file only after confirming that both `.efi`
entries are present.

## Verify the EFI files and random-seed permissions

```bash
ls -lh /boot/EFI/systemd/systemd-bootx64.efi
ls -lh /boot/EFI/BOOT/BOOTX64.EFI
findmnt -no OPTIONS /boot
stat -c '%a %U:%G %n' /boot/loader/random-seed
```

The mount options must include `fmask=0177,dmask=0077`, as configured in
chapter 5. The random-seed file should therefore appear as mode `600` and be
owned by `root:root`. FAT does not store Unix permissions; these values are
provided by the mount masks.

Do not continue if `bootctl` warns that the mount point or random-seed file is
world-accessible. Recheck the `/boot` record in `/etc/fstab` and its active
mount options first.

## Enable boot-manager updates

```bash
systemctl enable systemd-boot-update.service
systemctl is-enabled systemd-boot-update.service
```

The final command must print `enabled`. Do not use `--now` inside the chroot.
On future boots the service keeps the copies on the EFI System Partition in
sync with the systemd-boot binary supplied by the installed `systemd` package.
Chapter 13 will make this update path compatible with Secure Boot signatures.

## Verify the firmware entry

```bash
efibootmgr -v
lsblk -no PARTUUID /dev/nvme0n1p1
```

Confirm that a `Linux Boot Manager` entry points to:

```text
\EFI\systemd\systemd-bootx64.efi
```

Its partition identifier must match the PARTUUID printed for
`/dev/nvme0n1p1`.

If the entry is missing, create it explicitly and inspect the list again:

```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Linux Boot Manager" --loader '\EFI\systemd\systemd-bootx64.efi'
efibootmgr -v
```

Do not delete Fedora, Windows, old Linux Boot Manager, or other unfamiliar EFI
entries during this installation. Stale-entry identification and cleanup
belong in a separate maintenance guide; deleting the wrong entry can make
another installed system or disk harder to boot.

## Completion checkpoint

```bash
cat /boot/loader/loader.conf
bootctl --esp-path=/boot list
efibootmgr -v
systemctl is-enabled systemd-boot-update.service
```

Confirm that:

- the normal and fallback UKIs appear as Type #2 entries;
- `arch-linux.efi` is the default;
- systemd-boot exists at its normal and fallback EFI paths;
- `Linux Boot Manager` points to the current EFI System Partition;
- the update service is enabled.

The boot manager and UKIs are still unsigned. Do not enable Secure Boot or
reboot yet. Remain inside the chroot.

## Sources

- [ArchWiki: systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [Arch manual: bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [Arch manual: systemd-boot(7)](https://man.archlinux.org/man/systemd-boot.7)
- [Arch manual: loader.conf(5)](https://man.archlinux.org/man/loader.conf.5)
- [Arch manual: efibootmgr(8)](https://man.archlinux.org/man/efibootmgr.8)
- [Boot Loader Specification](https://uapi-group.org/specifications/specs/boot_loader_specification)

## Next step

Continue with chapter 13 to create and enroll Secure Boot keys, sign every EFI
binary in the boot chain, and verify the resulting signatures.
