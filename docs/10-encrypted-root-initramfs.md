# 10 — Encrypted-root initramfs

## Goal

Configure the early userspace that will unlock the LUKS2 container, activate
the LVM volume group, and mount the ext4 root filesystem during boot.

This chapter prepares `mkinitcpio` and `/etc/kernel/cmdline`. It does not build
the final images yet; chapter 11 will generate the normal and fallback UKIs
directly.

## Prerequisites

- Chapter 09 has been completed successfully.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- `/dev/nvme0n1p2` is the LUKS2 partition opened as `cryptlvm`.
- The root LV is `/dev/vg0/root`.
- `mkinitcpio`, `cryptsetup`, `lvm2`, and `amd-ucode` are installed.

## Configure the mkinitcpio hooks

Open the configuration file:

```bash
micro /etc/mkinitcpio.conf
```

Replace only the active `HOOKS=` line with:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

Save the file and close Micro. Leave `MODULES`, `BINARIES`, `FILES`, and the
compression settings unchanged unless the target hardware has a documented
reason to override them.

The critical relationships are:

| Hook | Purpose |
| --- | --- |
| `systemd` | Use the systemd-based initramfs rather than the BusyBox path. |
| `keyboard` and `sd-vconsole` | Provide keyboard input and the selected console keymap before root is mounted. |
| `sd-encrypt` | Discover and unlock the LUKS2 container. |
| `lvm2` | Activate `vg0` after the container is open. |
| `filesystems` and `fsck` | Check and mount the ext4 root filesystem. |

Hook order matters. Do not mix the systemd `sd-encrypt` hook with the
BusyBox-oriented `encrypt` hook.

Check the resulting line:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
```

It must match the line shown above.

## Read the LUKS UUID

The kernel command line must use the LUKS UUID created on this installation,
not a UUID copied from another computer or from this repository.

Capture and display it:

```bash
luks_uuid=$(cryptsetup luksUUID /dev/nvme0n1p2)
printf '%s\n' "$luks_uuid"
```

Continue only if the command prints one complete UUID. This is the UUID of the
LUKS2 container itself, not the partition `PARTUUID`, ext4 UUID, or LVM UUID.

## Create the kernel command line

```bash
printf 'rd.luks.name=%s=cryptlvm root=/dev/mapper/vg0-root rw\n' "$luks_uuid" > /etc/kernel/cmdline
```

The three parameters have distinct jobs:

| Parameter | Purpose |
| --- | --- |
| `rd.luks.name=<UUID>=cryptlvm` | Unlock that LUKS UUID as `/dev/mapper/cryptlvm`. |
| `root=/dev/mapper/vg0-root` | Select the root LV activated inside the container. |
| `rw` | Mount root read-write after the early filesystem check. |

`rd.luks.name=` already identifies and names the encrypted device, so a
separate `rd.luks.uuid=` parameter is unnecessary. Do not use
`cryptdevice=UUID=...`; that syntax belongs to the BusyBox `encrypt` hook, not
the systemd `sd-encrypt` path used here.

The command line deliberately omits:

- `resume=`, because hibernation is not configured;
- LUKS discard options, because discard propagation is a post-install choice;
- `quiet` and splash options, so early boot remains visible while the
  installation is being validated.

## Review the configuration

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cryptsetup luksUUID /dev/nvme0n1p2
ls -l /dev/mapper/cryptlvm /dev/mapper/vg0-root
```

Visually confirm that:

- the hook list contains `systemd`, `sd-encrypt`, and `lvm2` in the documented
  order;
- the UUID after `rd.luks.name=` exactly matches the UUID printed by
  `cryptsetup`;
- the mapping is named `cryptlvm`;
- the root path is `/dev/mapper/vg0-root`;
- both mapper paths currently exist.

Do not run `mkinitcpio -P` yet. The kernel preset still describes the temporary
artifacts created during `pacstrap`; chapter 11 will replace that preset and
then build both final UKIs in one operation.

Remain inside the chroot.

## Sources

- [ArchWiki: dm-crypt system configuration](https://wiki.archlinux.org/title/Dm-crypt/System_configuration)
- [ArchWiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
- [Arch manual: mkinitcpio(8)](https://man.archlinux.org/man/mkinitcpio.8)
- [Arch manual: mkinitcpio.conf(5)](https://man.archlinux.org/man/mkinitcpio.conf.5)
- [Arch manual: cryptsetup-luksUUID(8)](https://man.archlinux.org/man/cryptsetup-luksUUID.8)
- [Arch manual: systemd-cryptsetup-generator(8)](https://man.archlinux.org/man/systemd-cryptsetup-generator.8)
- [Arch manual: kernel-command-line(7)](https://man.archlinux.org/man/kernel-command-line.7)

## Next step

Continue with chapter 11 to configure the Linux kernel preset for direct UKI
output and generate normal plus fallback unified kernel images.
