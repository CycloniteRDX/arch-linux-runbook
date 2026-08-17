# 08 — Basic system configuration

## Goal

Enter the installed system and configure its time zone, hardware clock,
locale, console keyboard, hostname, and root password.

The canonical profile uses:

| Setting | Value |
| --- | --- |
| Time zone | `Europe/Madrid` |
| Hardware clock | UTC |
| System language | English: `en_US.UTF-8` |
| Console keyboard | US: `us` |
| Example hostname | `rogue-t14-r5` |

Spanish alternatives are included for the second ThinkPad. The system
language and keyboard layout are independent choices: using
`LANG=en_US.UTF-8` together with `KEYMAP=es` is valid.

## Prerequisites

- Chapter 07 has been completed successfully.
- Root, home, and the ESP remain mounted below `/mnt`.
- `/mnt/etc/fstab` contains the four validated records.
- The live environment's clock is correct.

## Enter the installed system

```bash
arch-chroot /mnt
```

Every following command in this chapter runs as `root` inside the chroot. Do
not prefix these commands with `sudo`.

## Configure the time zone and hardware clock

```bash
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc
```

`hwclock --systohc` copies the current system time to the hardware clock and
records UTC in `/etc/adjtime`.

Check the result:

```bash
date
readlink -f /etc/localtime
tail -n 1 /etc/adjtime
```

The link must resolve to `/usr/share/zoneinfo/Europe/Madrid`, and the final
line of `/etc/adjtime` must be `UTC`.

## Generate the required locales

Open the locale list:

```bash
micro /etc/locale.gen
```

Uncomment this line for the canonical English system:

```text
en_US.UTF-8 UTF-8
```

Also uncomment the following line if the Spanish system locale may be used:

```text
es_ES.UTF-8 UTF-8
```

Save the file, exit the editor, and generate every uncommented locale:

```bash
locale-gen
```

## Select the system language

For the canonical English system, create `/etc/locale.conf` with:

```bash
printf 'LANG=en_US.UTF-8\n' > /etc/locale.conf
```

To use Spanish system messages instead, run this command **in place of** the
previous one:

```bash
printf 'LANG=es_ES.UTF-8\n' > /etc/locale.conf
```

Generating both locales is harmless; the value in `/etc/locale.conf` selects
the default used by future sessions.

## Select the console keyboard

For a physical US keyboard, create `/etc/vconsole.conf` with:

```bash
printf 'KEYMAP=us\n' > /etc/vconsole.conf
```

For a physical Spanish keyboard, run this command **in place of** the previous
one:

```bash
printf 'KEYMAP=es\n' > /etc/vconsole.conf
```

Choose the keymap that matches the laptop's keyboard, especially if the LUKS
passphrase contains symbols. The graphical Niri keymap will be configured
later in the dotfiles repository.

## Configure a unique hostname

The following example identifies the Ryzen 5 ThinkPad:

```bash
printf 'rogue-t14-r5\n' > /etc/hostname
```

Replace `rogue-t14-r5` before running the command when installing another
machine. Use a unique lowercase name containing only letters, digits, and
hyphens; for example, the other laptop could use `rogue-t14-r7`.

No manual `/etc/hosts` entry is required for this single-machine setup. The
base Arch configuration resolves the local static hostname through systemd's
`myhostname` NSS module.

## Set the root password

```bash
passwd
```

Enter the new password twice when prompted. This password belongs to the root
account and is separate from the LUKS passphrase.

## Review the configuration

```bash
cat /etc/locale.conf
cat /etc/vconsole.conf
cat /etc/hostname
readlink -f /etc/localtime
tail -n 1 /etc/adjtime
passwd -S root
```

Confirm that the three text files contain the selected values, the time-zone
link resolves to Madrid, the hardware clock mode is `UTC`, and the root status
line contains `P`, meaning that the account has a password.

Do not exit the chroot yet. The next chapters continue working inside it.

## Sources

- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: arch-chroot(8)](https://man.archlinux.org/man/arch-chroot.8)
- [Arch manual: hwclock(8)](https://man.archlinux.org/man/hwclock.8)
- [ArchWiki: Locale](https://wiki.archlinux.org/title/Locale)
- [Arch manual: locale.conf(5)](https://man.archlinux.org/man/locale.conf.5)
- [Arch manual: vconsole.conf(5)](https://man.archlinux.org/man/vconsole.conf.5)
- [Arch manual: hostname(5)](https://man.archlinux.org/man/hostname.5)
- [Arch manual: nss-myhostname(8)](https://man.archlinux.org/man/nss-myhostname.8)
- [Arch manual: passwd(1)](https://man.archlinux.org/man/passwd.1)

## Next step

Continue with chapter 09 to create the regular user, grant controlled
administrative access with `sudo`, and enable NetworkManager.
