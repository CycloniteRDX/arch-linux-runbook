# 11 — Unified kernel images

## Goal

Configure the Linux kernel preset to generate two unified kernel images
directly:

| Image | Purpose |
| --- | --- |
| `arch-linux.efi` | Normal image optimized for the detected hardware. |
| `arch-linux-fallback.efi` | Broader recovery image built without the `autodetect` hook. |

Each UKI combines the EFI stub, kernel, AMD microcode, initramfs, operating
system metadata, and the kernel command line into one EFI executable.

The fallback image uses the same kernel and configuration as the normal image;
it merely contains a broader set of modules. It is not a rollback to an older
kernel.

## Prerequisites

- Chapter 10 has been completed successfully.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- `/boot` is the mounted EFI System Partition.
- `/etc/kernel/cmdline` contains the verified LUKS and root parameters.
- `mkinitcpio`, `systemd-ukify`, and `amd-ucode` are installed.

Recheck the inputs before building:

```bash
findmnt /boot
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
```

Stop if `/boot` is not backed by `/dev/nvme0n1p1` or either configuration
differs from chapter 10.

## Create the UKI directory

Boot Loader Specification Type #2 images belong below `EFI/Linux` on the boot
partition:

```bash
mkdir -p /boot/EFI/Linux
```

## Configure the kernel preset

Open the preset installed for the standard Arch kernel:

```bash
micro /etc/mkinitcpio.d/linux.preset
```

Replace its contents with:

```bash
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

default_uki="/boot/EFI/Linux/arch-linux.efi"

fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Save and close Micro. The `_uki` destinations tell `mkinitcpio` to create UKIs
instead of separate initramfs files. Skipping `autodetect` only for the
fallback preset causes it to include a wider set of modules.

Review the complete file:

```bash
cat /etc/mkinitcpio.d/linux.preset
```

There must be no active `default_image=` or `fallback_image=` assignment.

## Generate both UKIs

```bash
mkinitcpio -P
```

`-P` processes both presets. For each one, the output should report a
successful initramfs build followed by UKI creation with `ukify`.

On a fresh installation the output may say that the UKI was written unsigned.
That is expected: the Secure Boot keys have not been created yet. The `sbctl`
post-hook safely skips signing when no keys are available; signing is completed
in chapter 13.

The fallback build may warn about missing firmware for hardware not present in
these ThinkPads. Read every warning, but distinguish it from an error that
aborts image generation. Both presets must finish successfully.

## Inspect the generated images

```bash
ls -lh /boot/EFI/Linux
bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
```

Both identification commands must print:

```text
uki
```

Inspect their embedded metadata:

```bash
bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

For both images, confirm that:

- the kernel type is `uki`;
- the operating system is Arch Linux;
- a kernel version is present;
- the embedded command line exactly matches `/etc/kernel/cmdline`;
- the fallback image is normally larger than the default image.

## Remove obsolete standalone initramfs files

The kernel installation performed by `pacstrap` may have created two temporary
standalone initramfs files before the UKI preset existed. After both UKIs have
been inspected successfully, list `/boot`:

```bash
ls -lh /boot
```

If these exact two files exist, remove them:

```bash
rm /boot/initramfs-linux.img
rm /boot/initramfs-linux-fallback.img
```

Do not remove `/boot/vmlinuz-linux`. It remains the package-managed kernel used
as input whenever `mkinitcpio` rebuilds the UKIs. It will not be used as a
standalone boot entry in the finished system.

## Completion checkpoint

```bash
ls -lh /boot/EFI/Linux
cat /etc/mkinitcpio.d/linux.preset
cat /etc/kernel/cmdline
```

The `EFI/Linux` directory must contain both UKIs and the preset must point to
those exact paths. The system still lacks a boot manager and Secure Boot
signatures, so do not reboot yet.

Remain inside the chroot.

## Sources

- [ArchWiki: Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [ArchWiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
- [Arch manual: mkinitcpio(8)](https://man.archlinux.org/man/mkinitcpio.8)
- [Arch manual: ukify(1)](https://man.archlinux.org/man/ukify.1)
- [Arch manual: systemd-stub(7)](https://man.archlinux.org/man/systemd-stub.7)
- [Arch manual: bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [sbctl upstream repository](https://github.com/Foxboron/sbctl)

## Next step

Continue with chapter 12 to install systemd-boot, configure its menu, and
verify that it discovers both UKIs automatically.
