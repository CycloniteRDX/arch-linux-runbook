# 01 — Pre-installation

## Goal

Boot the official Arch Linux installation medium and confirm that the live
environment, network, clock, and target disk are ready for installation.

Nothing in this chapter writes to the internal SSD.

## Prerequisites

Before booting the installer:

- Back up every file that must be preserved from the target laptop.
- Write a current official Arch Linux ISO to a USB drive.
- Connect the ThinkPad to AC power.
- Disconnect external drives that are not needed for the installation.
- Configure the firmware for UEFI-only boot.
- Temporarily disable Secure Boot so the official installation medium can
  start. Do not clear the firmware keys merely to boot the ISO.

This runbook erases the existing partition table and operating systems on the
selected internal SSD. It is not a dual-boot procedure.

## Boot the installation medium

1. Insert the Arch Linux USB drive.
2. Power on the ThinkPad and open the one-time boot menu with `F12`.
3. Select the USB entry explicitly marked as UEFI.
4. Start the default Arch Linux installation-medium entry.

Continue when the live environment displays a root shell similar to:

```text
root@archiso ~ #
```

Commands in the live environment are already executed as root. Do not add
`sudo` to the commands in this runbook.

## Set the temporary console keymap

The installation medium starts with a US console keymap.

List all available console keymaps if needed:

```bash
localectl list-keymaps
```

For a laptop with a US physical keyboard, no change is required. To select it
explicitly:

```bash
loadkeys us
```

For a laptop with a Spanish physical keyboard:

```bash
loadkeys es
```

This setting affects only the live environment. The installed system keymap is
configured later and may be selected independently from the system language.

## Optional console comfort

These changes affect only the current live session and are not required for the
installation.

### Use a larger console font

The Terminus `ter-132b` font is easier to read on a 14-inch Full HD display:

```bash
setfont ter-132b
```

To inspect other fonts included in the live environment:

```bash
ls /usr/share/kbd/consolefonts/
```

### Disable the console speaker beep

Unload the PC-speaker module:

```bash
modprobe -r pcspkr
```

The module and beep return after reboot unless the installed system is
configured separately.

## Verify UEFI mode

```bash
cat /sys/firmware/efi/fw_platform_size
```

Expected output on these ThinkPads:

```text
64
```

Stop if the file does not exist or the command does not print `64`. Reboot and
select the UEFI USB entry instead of a legacy or compatibility-mode entry.

## Connect to the network

Check whether a wireless device is blocked:

```bash
rfkill
```

If the wireless row reports a software-blocked state, remove the block and
check it again:

```bash
rfkill unblock wlan
rfkill
```

`rfkill unblock` cannot remove a hard block. If the device remains hard
blocked, use the laptop's wireless key or switch, or review its firmware
settings.

List the available network interfaces:

```bash
ip link
```

### Ethernet

For a supported wired adapter, connect the cable and continue to the connection
test below. The live environment obtains an address automatically.

### Wi-Fi

List the wireless device:

```bash
iwctl device list
```

The examples below assume that the device is named `wlan0`. Replace that name
if `iwctl` reports a different one.

```bash
iwctl station wlan0 scan
iwctl station wlan0 get-networks
iwctl station wlan0 connect "NETWORK_NAME"
```

`iwctl` requests the Wi-Fi passphrase interactively when required. Do not place
the passphrase directly in the command.

Verify both connectivity and DNS resolution:

```bash
ping -c 3 archlinux.org
```

Continue only after replies are received.

## Synchronize the clock

```bash
timedatectl set-ntp true
timedatectl
```

Confirm that the NTP service is active. Initial synchronization can take a
short time after the network connection is established.

## Identify the target disk

List whole disks with their model, serial number, size, and connection type:

```bash
lsblk -d -o NAME,PATH,SIZE,MODEL,SERIAL,TRAN
```

Then inspect partitions, filesystems, and mount points:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

For the canonical profile, the internal SSD is expected to be an NVMe device
named `/dev/nvme0n1` with a capacity of approximately `476.9G`. The USB drive,
loop device, and internal NVMe SSD must not be confused.

Inspect the current partition table without changing it:

```bash
fdisk -l /dev/nvme0n1
```

Existing partitions are expected when reinstalling a used laptop. They are not
removed in this chapter.

## Stop conditions

Do not continue to partitioning if any of the following is true:

- Required data has not been backed up.
- The environment is not booted in 64-bit UEFI mode.
- The installation medium has no working network connection.
- The system clock is clearly incorrect after NTP has had time to synchronize.
- `/dev/nvme0n1` does not match the expected internal SSD.
- More than one possible target disk remains connected.
- The computer must preserve an existing operating system or dual-boot layout.

## Expected result

Before continuing, all of these statements must be true:

- The prompt is the Arch Linux live root shell.
- The console keymap matches the physical keyboard.
- `fw_platform_size` reports `64`.
- `ping -c 3 archlinux.org` succeeds.
- NTP is active.
- The internal SSD has been identified by path, size, model, and serial number.
- The intended target is `/dev/nvme0n1`.

## Sources

- [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: iwctl(1)](https://man.archlinux.org/man/iwctl.1)
- [Arch manual: rfkill(8)](https://man.archlinux.org/man/rfkill.8)
- [Arch manual: loadkeys(1)](https://man.archlinux.org/man/loadkeys.1)
- [ArchWiki: Linux console fonts](https://wiki.archlinux.org/title/Linux_console#Fonts)
- [Arch manual: lsblk(8)](https://man.archlinux.org/man/lsblk.8)
- [Arch manual: fdisk(8)](https://man.archlinux.org/man/fdisk.8)
- [ArchWiki: Secure Boot installation media](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Booting_an_installation_medium)

## Next step

Continue with chapter 02 to replace the target disk's current partition table
with the canonical GPT layout.
