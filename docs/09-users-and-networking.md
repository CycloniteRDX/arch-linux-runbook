# 09 — Users and networking

## Goal

Create the regular user, grant password-protected administrative access, and
enable the services required for networking and automatic time
synchronization after the first boot.

The canonical user is `neon`. This chapter grants administrative access only
through the `wheel` group and keeps the root account available for recovery.

## Prerequisites

- Chapter 08 has been completed successfully.
- The shell is still running as `root` inside `arch-chroot /mnt`.
- The `sudo` and `networkmanager` packages are installed.
- The root account has a working password.

Do not exit the root chroot shell until the new user's `sudo` access has been
tested successfully.

## Create the regular user

```bash
useradd -m -U -G wheel -s /bin/bash neon
passwd neon
```

The options create `/home/neon`, create a matching primary group, add the user
to the supplementary `wheel` group, and select Bash as the login shell.

Enter the user's password twice when prompted. Do not reuse the LUKS
passphrase or place any password directly in a command.

Check the account:

```bash
id neon
getent passwd neon
ls -ld /home/neon
passwd -S neon
```

Confirm that:

- the account belongs to the `neon` and `wheel` groups;
- its home is `/home/neon` and is owned by `neon`;
- its login shell is `/bin/bash`;
- the password status contains `P`.

Do not add the user to broad groups such as `disk`, `storage`, `audio`,
`video`, or `input`. Modern desktop sessions normally receive device access
through udev and systemd-logind. Add another group later only when a specific
device or application requires it.

## Configure sudo safely

Create a dedicated sudoers drop-in with `visudo`:

```bash
EDITOR=micro visudo -f /etc/sudoers.d/10-wheel
```

Enter exactly this one line:

```text
%wheel ALL=(ALL:ALL) ALL
```

Save and close Micro. `visudo` checks the file before installing it. The rule
allows members of `wheel` to run commands as other users, including root,
after authenticating with their own password.

Set the required permissions and validate the complete sudoers policy:

```bash
chmod 440 /etc/sudoers.d/10-wheel
visudo -c
```

Every reported sudoers file must parse successfully. Do not use `NOPASSWD` for
the daily administrative account.

## Test the administrative path

Open a login shell as the new user without closing the root shell:

```bash
su - neon
```

Then force a password prompt and test privilege escalation:

```bash
sudo -k
sudo whoami
```

Enter the password assigned to `neon`. The final command must print:

```text
root
```

Return to the root chroot shell:

```bash
exit
```

Stop and repair the sudoers configuration if the test fails. The existing root
shell provides a safe recovery path.

## Enable networking for the first boot

```bash
systemctl enable NetworkManager.service
```

Do not add `--now`: the installed system's service manager is not running
inside the chroot. Enabling the unit prepares it to start during the first real
boot.

The Wi-Fi connection used by the installation image is not copied into the
installed system. The first-boot chapter will reconnect using NetworkManager.
Wired Ethernet normally obtains a connection automatically through DHCP.

Polkit is deliberately deferred until the graphical workstation is assembled.
The first Wi-Fi profile is therefore created with `sudo nmcli` in chapter 14;
read-only `nmcli` inspection remains available to the regular user.

## Enable automatic time synchronization

```bash
systemctl enable systemd-timesyncd.service
```

`systemd-timesyncd` is supplied by systemd and is sufficient for this laptop.
It will synchronize the clock after networking becomes available. Do not run
`timedatectl set-ntp true` inside the chroot because there is no running systemd
instance there to service the request.

## Review the configuration

```bash
id neon
getent passwd neon
ls -ld /home/neon
passwd -S neon
visudo -c
systemctl is-enabled NetworkManager.service
systemctl is-enabled systemd-timesyncd.service
```

Both service checks must print `enabled`, and the sudoers policy must remain
valid. The target now has a regular administrative user and the services needed
for network access and clock synchronization after boot.

Remain inside the chroot for the next chapter.

## Sources

- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [ArchWiki: Users and groups](https://wiki.archlinux.org/title/Users_and_groups)
- [ArchWiki: sudo](https://wiki.archlinux.org/title/Sudo)
- [Arch manual: useradd(8)](https://man.archlinux.org/man/useradd.8)
- [Arch manual: visudo(8)](https://man.archlinux.org/man/visudo.8)
- [Arch manual: sudoers(5)](https://man.archlinux.org/man/sudoers.5)
- [Arch manual: NetworkManager(8)](https://man.archlinux.org/man/NetworkManager.8)
- [Arch manual: systemctl(1)](https://man.archlinux.org/man/systemctl.1)
- [Arch manual: systemd-timesyncd(8)](https://man.archlinux.org/man/systemd-timesyncd.service.8)

## Next step

Continue with chapter 10 to configure the systemd-based mkinitcpio hooks and
the kernel command line required to unlock LUKS2 and activate the LVM root
volume during early boot.
