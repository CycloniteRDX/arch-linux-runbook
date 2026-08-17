# 13 — Secure Boot

## Goal

Create owner-controlled Secure Boot keys, enroll them together with Microsoft
vendor certificates, and sign every EFI executable used by the canonical boot
chain.

The completed chain is:

1. UEFI firmware trusts the owner and Microsoft certificates.
2. The firmware verifies signed systemd-boot.
3. systemd-boot starts a signed normal or fallback UKI.
4. The UKI authenticates its embedded kernel, microcode, initramfs, and command
   line as one EFI executable.

## Prerequisites

- Chapter 12 has been completed successfully.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- The firmware was put in Setup Mode before booting the installation medium.
- `/boot` is the mounted EFI System Partition with root-only mount masks.
- Both UKIs and all systemd-boot copies from chapter 12 exist.
- `sbctl` is installed.

Check the current state:

```bash
sbctl status
findmnt /boot
bootctl --esp-path=/boot list
```

Before creating keys, `sbctl status` must report that Setup Mode is enabled and
Secure Boot is disabled. Stop if Setup Mode is disabled; the firmware will
reject owner-key enrollment.

## Create the owner keys

```bash
sbctl create-keys
sbctl status
```

`sbctl` creates a Platform Key, Key Exchange Key, and Signature Database key
below `/var/lib/sbctl/keys`, together with an owner GUID. These private keys
must never be copied to the EFI System Partition or committed to a repository.

## Enroll owner and Microsoft certificates

> [!CAUTION]
> Incorrect Secure Boot enrollment can prevent firmware components, expansion
> devices, external media, or operating systems from starting. The canonical
> command retains Microsoft vendor certificates for compatibility with signed
> Option ROMs and recovery media.

```bash
sbctl enroll-keys --microsoft
sbctl status
```

The enrollment must complete without errors. Setup Mode should normally become
disabled because a new Platform Key exists, although some firmware does not
refresh the displayed state until reboot. The Secure Boot line may likewise
not show the final enabled state yet; chapter 14 verifies both from the
installed system.

Do not use `sbctl enroll-keys` without `--microsoft` on this hardware profile.
Never bypass an Option ROM warning with `--yes-this-might-brick-my-machine`.

If enrollment fails only because EFI variables are marked immutable, retry
with sbctl's dedicated handling instead of changing attributes manually:

```bash
sbctl enroll-keys --microsoft --ignore-immutable
```

Use that option only for the specific immutable-variable error. Stop for every
other enrollment failure.

## Prepare signed systemd-boot updates

First create and register a signed copy beside the package-managed bootloader:

```bash
sbctl sign --save --output /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```

`bootctl install` and `bootctl update` prefer the `.efi.signed` file when it
exists. Reinstall systemd-boot now so the normal and UEFI fallback locations
receive that signed binary immediately:

```bash
bootctl --esp-path=/boot install
```

The copies on the EFI System Partition do not need separate database records.
The saved source record keeps `.efi.signed` current, while `bootctl` distributes
that signed output to every installed systemd-boot location.

## Sign both UKIs

```bash
sbctl sign --save /boot/EFI/Linux/arch-linux.efi
sbctl sign --save /boot/EFI/Linux/arch-linux-fallback.efi
```

The standalone `/boot/vmlinuz-linux` file is package-managed input for
`mkinitcpio`; no boot entry executes it directly. The signed UKIs are the
actual kernel artifacts in this installation, so `vmlinuz-linux` is not added
to the signing database.

## Test automatic UKI signing

Rebuild both UKIs once after creating the keys:

```bash
mkinitcpio -P
```

Each preset must complete successfully. Its post-processing output should show
the `sbctl` hook signing the newly generated UKI. This confirms that future
kernel, microcode, or initramfs updates will not leave the UKIs unsigned.

## Verify the signing database and EFI files

```bash
sbctl list-files
sbctl verify
```

Confirm that the saved paths include:

```text
/usr/lib/systemd/boot/efi/systemd-bootx64.efi
/boot/EFI/Linux/arch-linux.efi
/boot/EFI/Linux/arch-linux-fallback.efi
```

The first record should show
`/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed` as its output file.

`sbctl verify` must mark all systemd-boot executables and both UKIs as signed.
It may also report that `/boot/vmlinuz-linux` is unsigned. That is expected and
does not create an unsigned boot path because `bootctl list` contains no entry
that references it.

Inspect both UKIs one last time:

```bash
bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Their embedded command lines must still match `/etc/kernel/cmdline`.

## Completion checkpoint

```bash
sbctl status
sbctl verify
bootctl --esp-path=/boot list
systemctl is-enabled systemd-boot-update.service
```

Confirm that:

- owner keys exist and enrollment completed successfully;
- Microsoft vendor certificates were included during enrollment;
- both UKIs are signed and listed as Type #2 entries;
- all installed systemd-boot executables are signed;
- `arch-linux.efi` remains the default entry;
- `systemd-boot-update.service` remains enabled.

Do not delete `/var/lib/sbctl`. It contains the private keys needed to sign
future EFI binaries. An encrypted offline backup of these keys should be made
after the first successful boot and documented outside this installation
runbook.

Do not reboot until every required EFI executable passes verification.
Remain inside the chroot.

## Sources

- [ArchWiki: Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Arch manual: sbctl(8)](https://man.archlinux.org/man/sbctl.8)
- [Arch manual: bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [ArchWiki: Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [sbctl upstream repository](https://github.com/Foxboron/sbctl)

## Next step

Continue with chapter 14 to leave the chroot, unmount the installation safely,
perform the first Secure Boot start, unlock LUKS, and verify the TTY system.
