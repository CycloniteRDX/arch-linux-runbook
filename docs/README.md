# Runbook chapter map

The runbook will be written and tested one chapter at a time. Planned chapters
are listed below; filenames may change before the first stable release.

| Chapter | Purpose |
| --- | --- |
| [00. Installation profile](00-installation-profile.md) | Define assumptions, substitutions, and the final layout. |
| [01. Pre-installation](01-pre-installation.md) | Boot the ISO, verify UEFI mode, connect to the network, and confirm the target disk. |
| [02. Partitioning](02-partitioning.md) | Create the GPT layout and preserve the intentional unallocated SSD space. |
| [03. LUKS2 encryption](03-luks2-encryption.md) | Create and open the encrypted container. |
| [04. LVM and filesystems](04-lvm-and-filesystems.md) | Create the volume group, logical volumes, swap, and ext4 filesystems. |
| [05. Mount target](05-mount-target.md) | Mount `/`, `/home`, and `/boot`, then enable installation swap. |
| [06. Base system](06-install-base-system.md) | Install the base packages, kernel, firmware, microcode, networking, and essential tools. |
| 07. Generate fstab | Generate the file once with `>`, inspect it with `cat`, and verify it with `findmnt`. |
| 08. Basic system configuration | Configure time, locale, console keymap, hostname, hosts, root access, and chroot. |
| 09. Users and networking | Create the regular user, configure `sudo`, and enable NetworkManager. |
| 10. Encrypted-root initramfs | Configure the systemd-based `mkinitcpio` hooks and kernel command line. |
| 11. Unified kernel images | Generate normal and fallback UKIs directly from the kernel preset. |
| 12. systemd-boot | Install and configure the boot manager around the UKIs. |
| 13. Secure Boot | Create or enroll keys safely, sign required EFI binaries, and verify signatures. |
| 14. First boot | Exit the chroot, unmount safely, reboot, and verify LUKS unlock plus TTY login. |

## Boundary conditions

The following subjects are deliberately excluded from this sequence:

- Reflector as a persistent mirror-management service.
- Periodic TRIM and encrypted discard policy.
- Zram and memory tuning.
- Hibernation.
- Firewall and broader hardening.
- Laptop power management.
- Audio, Bluetooth, printing, fonts, and user directories.
- Niri and graphical applications.

Those decisions begin in the post-install repository. Their underlying theory
belongs in the handbook.
