# 14 — First boot

## Goal

Close the installation environment safely, start the new system through the
signed boot chain, unlock LUKS, log in as the regular user, and verify the
canonical installation from its first TTY.

Reaching a login prompt is necessary but not sufficient. This chapter also
checks that the expected UKI, encrypted mapping, filesystems, swap, services,
network, and Secure Boot state are actually in use.

## Prerequisites

- Chapter 13 has been completed without enrollment or signing errors.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- Both UKIs and every systemd-boot executable required by the boot chain are
  signed.
- `arch-linux.efi` is the default systemd-boot entry.
- The regular user and root passwords are known.
- The LUKS passphrase is known.

Perform one final check before leaving the chroot:

```bash
sbctl verify
bootctl --esp-path=/boot list
findmnt /boot
```

Do not reboot if a required systemd-boot executable or either UKI is unsigned.
The expected unsigned `/boot/vmlinuz-linux` result described in chapter 13 is
not a boot-chain failure.

## Leave the chroot

Flush pending writes and return to the installation-medium shell:

```bash
sync
exit
```

The prompt should again belong to `root@archiso`. The following commands are
executed from the live installation environment, not from the chroot.

## Close the installed storage stack

Disable only the installed system's swap logical volume:

```bash
swapoff /dev/vg0/swap
swapon --show
```

The second command must no longer list `/dev/vg0/swap`. It may still show swap
provided internally by the installation medium.

Unmount the complete target hierarchy:

```bash
umount -R /mnt
findmnt -R /mnt
```

`findmnt` should print nothing. If unmounting reports that a target is busy, do
not use a force or lazy unmount. Close any shell or process still using `/mnt`
and retry.

Deactivate the volume group and close the LUKS mapping:

```bash
vgchange -an vg0
cryptsetup close cryptlvm
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

The final tree should show `/dev/nvme0n1p2` as `crypto_LUKS`, without the
`cryptlvm` mapping or its logical volumes beneath it. Closing the mapping also
removes its encryption key from kernel memory.

## Reboot into the firmware

The tested ThinkPads support requesting the firmware setup screen directly:

```bash
systemctl reboot --firmware-setup
```

If the command reports that the feature is unsupported, run `reboot` instead
and press `F1` when the Lenovo logo appears.

## Verify the firmware state

In the ThinkPad UEFI setup utility:

1. Open `Security` → `Secure Boot`.
2. Confirm that `Platform Mode` is `User Mode`, not `Setup Mode`.
3. Confirm that Secure Boot is enabled.
4. Leave the custom Secure Boot key state unchanged.
5. Confirm that `Linux Boot Manager` is ahead of obsolete disk entries in the
   boot order.
6. Remove the Arch Linux installation USB drive.
7. Save with `F10` and restart.

Do not select `Restore Factory Keys`, `Reset to Setup Mode`, or `Clear All
Secure Boot Keys`. Any of those operations would replace or remove trust in
the owner keys enrolled by `sbctl`.

If Secure Boot remains disabled but Platform Mode is already `User Mode`,
enable Secure Boot without restoring the factory keys. If Platform Mode still
reports `Setup Mode`, do not create another key set: boot the installed system
with Secure Boot disabled and inspect the earlier enrollment before retrying.

## Start the installed system

systemd-boot should display the normal and fallback UKIs. Allow the default
`arch-linux.efi` entry to start.

Enter the LUKS passphrase when requested. A successful unlock activates `vg0`,
mounts the root and home logical volumes, and continues to the TTY login
prompt.

Log in as:

```text
neon
```

The fallback UKI contains a broader module set but the same kernel version. If
the normal UKI fails before reaching the LUKS prompt, reboot and select
`arch-linux-fallback.efi` once. Treat success with only the fallback image as a
diagnostic result, not as completion of the runbook.

## Verify Secure Boot and the active boot path

```bash
sudo sbctl status
sudo sbctl verify
sudo bootctl status
sudo bootctl list
sudo bootctl --print-loader-path
```

Confirm that:

- Setup Mode is disabled;
- Secure Boot is enabled;
- the enrolled vendor-key line includes Microsoft;
- the current and default entry is `arch-linux.efi`;
- the loader path is `/boot/EFI/systemd/systemd-bootx64.efi`;
- both UKIs and the systemd-boot executables are signed.

If the loader path is `/boot/EFI/BOOT/BOOTX64.EFI`, the signed fallback path
worked but the firmware did not use the intended `Linux Boot Manager` entry.
The installation is bootable, but the firmware entry should be repaired before
declaring the runbook test complete.

As before, an unsigned `/boot/vmlinuz-linux` report is expected because no boot
entry executes it directly.

## Verify the encrypted root and mounted filesystems

```bash
cat /proc/cmdline
sudo cryptsetup status cryptlvm
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
findmnt /
findmnt /home
findmnt /boot
swapon --show
```

Confirm that:

- the active command line contains the expected `rd.luks.name=` and `root=`
  parameters;
- `cryptlvm` is an active LUKS2 mapping backed by `/dev/nvme0n1p2`;
- `/` uses `/dev/mapper/vg0-root` as ext4;
- `/home` uses `/dev/mapper/vg0-home` as ext4;
- `/boot` uses `/dev/nvme0n1p1` as vfat;
- `/dev/mapper/vg0-swap` is active swap.

Verify that the EFI System Partition retained its root-only mount masks:

```bash
findmnt -no OPTIONS /boot
sudo stat -c '%a %U:%G %n' /boot/loader/random-seed
```

The options must include `fmask=0177,dmask=0077`, and the random-seed file
should appear as mode `600` owned by `root:root`.

## Verify identity, privileges, and services

```bash
hostnamectl
localectl
timedatectl
id
sudo -k
sudo whoami
systemctl is-enabled NetworkManager.service systemd-timesyncd.service systemd-boot-update.service
systemctl is-active NetworkManager.service systemd-timesyncd.service
systemctl --failed
```

Enter the regular user's password when `sudo` asks for it. Confirm that:

- the configured hostname, locale, and console keymap are correct;
- the logged-in account is `neon` with `wheel` membership;
- `sudo whoami` prints `root`;
- all three services are enabled;
- NetworkManager and systemd-timesyncd are active;
- no system units are in a failed state.

The clock may not be synchronized until a network connection is available.

## Connect with NetworkManager

Inspect the devices and currently active connections:

```bash
nmcli device status
nmcli connection show --active
```

Wired Ethernet should normally connect automatically. For Wi-Fi, list the
networks and request credentials interactively:

```bash
nmcli device wifi list
nmcli --ask device wifi connect "NETWORK_NAME"
```

Do not place the Wi-Fi password directly in the command. Verify connectivity,
DNS, and the clock:

```bash
ping -c 3 archlinux.org
timedatectl
```

Allow a short time for synchronization. `timedatectl` should eventually report
an active NTP service and a synchronized system clock.

## If the first boot fails

Use the failure point to identify the affected layer:

| Failure point | First action |
| --- | --- |
| Firmware reports a security violation before the menu | Temporarily disable Secure Boot, boot the ISO, return to the chroot, and inspect `sbctl status` plus `sbctl verify`. Do not reset or regenerate the keys. |
| systemd-boot opens but rejects one UKI | Try the signed fallback UKI once, then inspect both UKI signatures from the ISO. |
| Neither UKI reaches the LUKS prompt | Recheck `/etc/kernel/cmdline`, the mkinitcpio hooks, and both UKIs from the chroot. |
| LUKS unlocks but the root volume is not found | Recheck the `root=` mapper name and LVM hook from chapters 10 and 11. |
| TTY works but networking does not | Inspect `nmcli device status` and `systemctl status NetworkManager.service`. |

Do not respond to an initial failure by clearing firmware keys, formatting the
disk, regenerating the LUKS container, or reinstalling. The signed fallback
path and the installation medium provide recovery routes without destroying
the completed installation.

## Completion criteria

The installation runbook has succeeded only when all of these are true:

- The firmware reports Secure Boot enabled in User Mode.
- The intended systemd-boot path and normal UKI are active and signed.
- The LUKS passphrase unlocks the system.
- Root, home, boot, and swap use the expected devices and formats.
- The regular user can log in and obtain password-protected sudo access.
- NetworkManager establishes a working connection.
- The clock synchronizes through systemd-timesyncd.
- No systemd units are failed.

Record any deviation found during a real installation before changing the
runbook. A successful test should include the exact hardware model, ISO date,
package versions, and any firmware-specific differences.

## Sources

- [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: systemctl(1)](https://man.archlinux.org/man/systemctl.1)
- [Arch manual: cryptsetup-close(8)](https://man.archlinux.org/man/cryptsetup-close.8)
- [Arch manual: vgchange(8)](https://man.archlinux.org/man/vgchange.8)
- [Arch manual: sbctl(8)](https://man.archlinux.org/man/sbctl.8)
- [Arch manual: bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [Arch manual: nmcli(1)](https://man.archlinux.org/man/nmcli.1)

## Next step

The installation runbook ends here. Continue in `arch-linux-post-install` to
configure updates, TRIM, zram, firewall policy, laptop integration, applications,
and the graphical workstation before applying the Niri dotfiles.
