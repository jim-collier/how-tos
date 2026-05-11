<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD012 -- No blank lines -->
<!-- markdownlint-disable MD033 -- No inline html -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# Downgrade Debian from Testing to Stable

<!-- TOC ignore:true -->
## Table of contents
<!-- TOC -->

- [Why](#why)
- [A warning before starting](#a-warning-before-starting)
- [Instructions](#instructions)
	- [Install the same version of the kernel on Testing host, that is the latest for Stable](#install-the-same-version-of-the-kernel-on-testing-host-that-is-the-latest-for-stable)
		- [Install the older kernel](#install-the-older-kernel)
		- [Uninstall the Testing kernel metapackage](#uninstall-the-testing-kernel-metapackage)
		- [Do the rest of the gruntwork](#do-the-rest-of-the-gruntwork)
	- [You might as well change from initramfs-tools to dracut while you're at it](#you-might-as-well-change-from-initramfs-tools-to-dracut-while-youre-at-it)
- [Troubleshooting](#troubleshooting)
- [Document history](#document-history)
- [Copyright and license](#copyright-and-license)

<!-- /TOC -->

## Why

Debian Testing is a fine distro, and quasi-"rolling" at that. However, there are drawbacks:

- It is last in line for security updates. Which at a time when AIs are finding zero-day exploits at record pace, is no longer very tenable.
	- One mitigation for this is to use Stable (or even Unstable) for security updates. But that introduces a greater risk of installing unresolvable dependency chains - and generally makes the inherent challenges of running Testing in the first place, worse.
- For a few months before and after a major release, running on Testing is just _pain_. More so if you didn't know the process was underway (which is not easy to know), and accidentally do a full update with, say, `apt get dist-upgrade`.
- Too often, proprietary drivers fall behind the kernel. Especially Nvidia and ZFS. Setting Grub or systemd-boot to always booting to the _second_ newest kernel if installed, helps a great deal, but not 100%. Be prepared for breakage.

Especially when grappling with the last point, you might be tempted to just "downgrade" to Stable.

This is not officially supported, and most recommendations warn that things will break.

## A warning before starting

While it's true that there can and almost certainly will be dependency problems during this process that temporarily break your system, if you do it in the right order (kernel first, Grub & EFI last), things are unlikely to permanently break - even for highly complex configurations with tons of packages installed.

But to mitigate potential problems, do try to run non-apt software as much as possible. The more software you have installed from the Debian repository, the exponentially harder it becomes to maintain a single unified system running a system-wide coherent set of dependencies.

Before downgrading, is the _best_ time to see if you can permanently minimize your dependency on the official Debian repository, for application images.

`flatpak` and `AppImages` are the top-two advisable choices.

- _The `nix` package manager can be installed on most or all other distros, and isolates dependencies similar to `flatpak` and `AppImages`. But the real magic of Nix is when it runs on a fully integrated NixOS, and manages the entire system. In standalone mode, those advantages are largely lost, and arguably does not justify the inherent downsides._

Benefits of running `flatpak` and/or `AppImage`applications:

- Each application is bundled with it's own dependencies, that have no effect on other applications or the system in general.

- The stable versions available are generally more up-to-date that even Debian Testing's versions - sometimes significantly so.

- `flatpak` has a very `apt`-like single-command update feature.

Drawbacks

- `flatpak` applications can have issues with the sandboxing feature. Applications that need tight system integration - e.g. web browsers or development IDEs with live debuggers - can have challenges that need manual tweaking.

	- _The issue with the web browser can be overcome by simply having one browser installed through `apt`, set as the default. Then just use your favorite `flatpak` browser as normal by launching it directly or via keyboard shortcut._

- `AppImages` don't have a native auto-update feature, nor do they install a menu icon themselves. But it is possible to achieve both of those features with a bit of extra work up-front.

_For the love of goodness, just don't fall for the `snap` trap. "Why" is beyond the scope of this, but many an Ubuntu user can tell you why._

__Before starting this usually painless process, just make _absolutely sure_ you have an alternate way to boot, in a way that will let you `chroot` into your WIP environment, in case things do go sideways__.

## Instructions

### Install the same version of the kernel on Testing host, that is the latest for Stable

#### Install the older kernel

1. On an updated 'Stable' host: `dpkg --list | grep 'linux-image'`

1. On the 'Testing' host to downgrade:

	~~~bash
	## Set $kVer to same major.minor version as latest on on Stable host
	sudo apt update
	kVer="6.12"  ## From 6.12.74+deb13+1
	apt search "linux-image-${kVer}"
	sudo apt install  linux-image-${kVer}
	~~~

1. Reboot: `( (nohup bash -c 'sleep 5; sudo reboot' &>/dev/null) & disown ); sleep 0.5; exit`

1. Interrupt boot and select that kernel you just installed.

#### Uninstall the Testing kernel metapackage

1. Uninstall the kernel metapackage: `sudo apt purge linux-image-amd64`

1. Uninstall all higher-numbered kernels than same version running as Stable.

1. Run: `sudo apt autoremove`

1. Remove leftover higher-numbered kernel-related directories and symlinks under:

	- `/usr/src`
	- `/lib/modules`

1. Make a note of what the current stable version is called. (Since 2011 they've all been character names from the "Toy Story" movies.) You'll be using it by name to be extra cautious (and in some cases for this process to even work), rather than the alias `stable`. As of 2026 it's `trixie`. Update accordingly.

#### Do the rest of the gruntwork

You have to do this all in one shot, without even letting the screensaver kick in. Don't let the power fail, or yourself die, until it's finished.

~~~bash
## Backup UEFI and grub config
[[ -d /boot/backup.old ]] && sudo rm -rf               /boot/backup.old
[[ -d /boot/backup     ]] && sudo mv     /boot/backup  /boot/backup.old
sudo mkdir -p /boot/backup/boot
sudo mkdir -p /boot/backup/efi/grub
sudo cp -r /boot/efi          /boot/backup/boot/efi.bak
sudo cp    /etc/default/grub  /boot/backup/etc/

## Edit source files under `/etc/apt/sources.list` and `/etc/apt/sources.d/*.list`, change `testing` to latest stable name.
sudo sed -i 's/testing/trixie/g; s/forky/trixie/g' /etc/apt/sources.list  &&  sudo nano /etc/apt/sources.list
ls -lA /etc/apt/sources.list.d/
	## Manually edit any active ones in there

## Pin stable (important to use latest stable release by name, rather than 'stable'.
echo -e "Package: *\nPin: release n=stable\nPin-Priority: 1001" | sudo tee /etc/apt/preferences.d/stable 1>/dev/null  &&  sudo nano /etc/apt/preferences.d/stable

## Perform upgrade (show first)
sudo apt update  &&  sudo apt -s full-upgrade | less
sudo apt full-upgrade

## Fix packages held back
apt list '?installed ?not(?archive(stable))'

## Downgrade packages manually
sudo apt install package=version

## Or purge problem packages
sudo apt purge package

## Clean up
sudo apt autoremove --purge
sudo apt clean

## Reboot
( (nohup bash -c 'sleep 5; sudo reboot' &>/dev/null) & disown ); sleep 0.5; exit

## Verify
cat /etc/debian_version
apt policy

## Grub2 and EFI: Force uninstall. These are mostly redundant in case one or more fails
sudo dpkg --remove --force-all grub-efi-amd64 grub-efi-amd64-bin grub-common shim-signed
dpkg --purge --force-all $(dpkg -l | grep -i grub | awk '{print $2}')
sudo apt purge 'grub*'
sudo aptitude remove 'grub*'  ## This can usually resolve dependency issues.

## Install downgraded grub.
sudo apt autoremove --purge  ## Look carefully at what it plans to do
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt update
sudo apt install --reinstall grub-efi-amd64 grub-common shim-signed
sudo grub-install
sudo update-grub

## Make sure Grub is configured the way you want
sudo nano /etc/default/grub

## Verify
grub-install --version
apt policy grub-common

## Reboot
( (nohup bash -c 'sleep 5; sudo reboot' &>/dev/null) & disown ); sleep 0.5; exit

## Unpin stable (comment out lines in the file)
sudo nano /etc/apt/preferences.d/stable

## Install latest stable kernel metapackages:
sudo apt update  &&  sudo apt install  linux-image-amd64  linux-headers-amd64

## Reboot again
( (nohup bash -c 'sleep 5; sudo reboot' &>/dev/null) & disown ); sleep 0.5; exit

## Run regular update process
~~~

### You might as well change from initramfs-tools to dracut while you're at it

If you don't have a complicated setup - e.g. no Luks system partition encryption and/or root-on-ZFS that already have complicated init scripts working fine - then you can probably easily change to dracut, which is more modern and will be the successor to initramfs-tools in the future for Debian.

It usually goes without a hitch.

~~~bash
## Install dracut and remove initramfs-tools at the same time
sudo apt install dracut

## For some reason dracut always installs mdadm, and it's not even
## a real dependency. So if you're not using it, you should remove it,
## as it may cause boot problems - and definitely will for
## root-on-ZFS setups, even if you do that later.
sudo apt remove mdadm

## Uninstall initramfs stragglers
sudo initramfs-tools initramfs-tools-core
sudo apt autoremove

## Create the 'initrd's (aka 'initramfs'es).
## This will be automatic with updates after this.
sudo dracut --regenerate-all --force

## Reboot
( (nohup bash -c 'sleep 5; sudo reboot' &>/dev/null) & disown ); sleep 0.5; exit
~~~

## Troubleshooting

~~~bash
sudo dpkg --configure -a
sudo aptitude install -f
sudo apt clean
sudo apt autoclean
~~~

## Document history

- 20260508: Created.

## Copyright and license

Copyright © 2026 Jim Collier (ID: 1cv◂‡Vᛦ)<br>
Licensed under GNU GPL v2 <https://www.gnu.org/licenses/gpl-2.0.html>. No warranty.