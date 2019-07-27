---
layout: post
title: "Install a Bare Minimal Manjaro Linux, in the Arch Way"
date: 2019-07-27
---

Recently I tried to install a minimal Manjaro Linux (i.e. a CLI system with only basic packages) on a USB stick for me to hack on and have fun with. [Manjaro-Architect](https://manjaro.org/download/architect/) seems to be what I want. It's a net installer with a lot of customization options so I can install my Linux in the way I want. Unfortunately Manjaro-Architect [has some serious problems](https://forum.manjaro.org/t/serious-errors-in-manjaro-architect/95568/22?u=amaikinono) recently, making it basically unuseable.

Luckily, we can still do this in the command line way. Since Manjaro is based on Arch Linux, so just open Arch's [installation guide](https://wiki.archlinux.org/index.php/installation_guide), be prepared to ask Google sensei a lot, and you are good to go.

Before we start I would like to answer some questions you may have:

- You want a minimal system, and you know the installation process of Arch Linux, so why not just use Arch?

  One can have various reasons to prefer Manjaro over Arch. I personally like that its packages are not as bleeding-edge as Arch, and you can use multiple kernels easily.

- What's the good of installing Manjaro with command line?

  Using the architect installer and using command line are maybe the only 2 ways to install a minimal Manjaro, and you can only do the latter if the former is broken. Besides, once you feel familiar with the process, you can write yourself a little script to help you install Manjaro everywhere.

- What's the point of being "minimal"? I think Manjaro should be a user-friendly desktop OS.

  Sometimes we do weird Unix stuff just because we can and it's fun :)

Ok, now let's get into it.

## Preparation

First you need to prepare the installation media. Any Manjaro flavor will do the job. I recommend you use a desktop version so you have graphical partitioning tools, a web browser to read this blog post, and multiple apps on one screen (which will be of great help). But if you have some weird graphics card on which a desktop version fails, then Manjaro-Architect is coming to rescue.

Pick your version [here](https://manjaro.org/download/) and follow the instructions to create a bootable USB stick.

Now boot into the live environment using that USB stick. The first thing you want to do is connecting to the internet. This should be easy on a desktop environment, and if you use Manjaro-Architect, you can use `nmtui` which is just as convenient.

Open your terminal emulator, remember the superuser password is "manjaro", and go on to the next step.

## Format the Disk and Mount the File Systems

`gparted` can be used in desktop environment, and `fdisk`, `parted`, `cfdisk` or `cgdisk` can be used in a text console. Find any detailed Arch Linux installation tutorial and it will teach you one of the tools. I want to install a UEFI system, and here is my layout:

- `/dev/sdX1`: 300 MiB, fat32, with `boot` and `efi` flags. Should be at least 260 MiB.
- `/dev/sdX2`: Remainder of the disk, ext4.

If you are not sure which is the disk to be partitioned, run `lsblk` and look into the information like capacity and number of partitions.

Command line tools may not actually do the formatting, so you need to do it yourself:

``` sh
# mkfs.fat -F32 /dev/sdX1
# mkfs.ext4 /dev/sdX2
```

If you are installing to a usb stick, you may want to create the ext4 partition without a journal. See [ArchWiki](https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key#Installation_tweaks) for details.

Now we need to mount the file systems:

``` sh
# mount /dev/sdX2 /mnt
$ mkdir -p /mnt/boot/efi
# mount /dev/sdX1 /mnt/boot/efi
```

Using `/mnt/boot` instead of `/mnt/boot/efi` should be fine if you do not need dual booting, but I don't know for sure, the architect installer told me this.

If you've created more partitions, you can mount them to extra directories, like `mount /dev/sdX3 /mnt/home`.

Now we have full access to `/dev/sdX`, let's install something to it.

## Install Base Packages

The packages will be downloaded from internet, so make sure you are using the fastest mirrors:

``` sh
# pacman-mirrors --fasttrack
```

Or you can specify your location:

``` sh
# pacman-mirrors -c Country_name
```

Avaliable countries are:

``` sh
# pacman-mirrors --country-list
```

Then install the base packages:

``` sh
$ basestrap /mnt base
```

`basestrap` is the Manjaro version of `pacstrap`.

## Configurations

### Fstab

Run

``` sh
# bash -c 'fstabgen /mnt >> /mnt/etc/fstab'
```

`fstabgen` is the Manjaro version of `genfstab`. Using the quoted command directly (as in Arch's guide) may not work, since the redirection is not running as root. We use a subshell running as root to do all things, so it will work correctly.

You can then edit `/mnt/etc/fstab` (using `nano` or other editors). I replaced all `relatime` with `noatime`.

### Chroot

Run

``` sh
# manjaro-chroot /mnt
```

Some terminal emulators may not handle backspace input correctly after chroot. If this happens, use `xterm` instead.

### Time Zone & Locale

Run the following command to set the time zone to, for example, Shanghai:

``` sh
# ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# hwclock --systohc
```

Look into `/usr/share/zoneinfo/` to find your nearest city.

If you dual boot Manjaro and Windows, they will have a time difference, and you can use local time in Manjaro:

``` sh
# timedatectl set-local-rtc 1 --adjust-system-clock
```

Or use UTC time in Windows. Search the internet for how to do this.

Edit `/etc/locale.gen` to set the locales:

``` sh
# nano /etc/locale.gen
```
Uncomment your needed locales and run

``` sh
# locale-gen
```

Take a look at `/etc/locale.conf`. If it doesn't set `LANG` variable correctly, edit it. For example:

```
LANG=en_US.UTF-8
```

### Hostname

Come up with a pretty name for your PC, put it in `/etc/hostname`. Then modify `/etc/hosts`:

``` sh
127.0.0.1	localhost.localdomain	localhost
::1		localhost.localdomain	localhost
127.0.1.1	myhostname.localdomain	myhostname
```

replace `myhostname` with your hostname.

### Users, Passwords, Sudo

Run

``` sh
# passwd
```

to set your superuser password.

Create a user for yourself:

``` sh
# useradd -m -G wheel myusername
```

Set a passwork for it:

``` sh
# passwd myusername
```

Now use `visudo` to edit the configuration file of `sudo`, it will see if you are making syntax errors so it's more safe than edit it directly. You can specify the editor:

``` sh
# EDITOR=nano visudo
```

Then find this line:

```
# %wheel ALL=(ALL)ALL
```

Uncomment it to make `sudo` avaliable to the `wheel` user group.

### Install Additional Packages

First is the kernel itself. Pick whatever you want. I use the current LTS kernel:

``` sh
# pacman -S linux419
```

It may warn you that `aic94xx` and `wd719x` firmwares are missing. Don't worry, these are for SAS/SCSI Disk Controllers in server hardware, and are unnecessary for most users.

*Note: If you are installing Manjaro to a USB stick, then before creating the initial RAM disk, you should edit `/etc/mkinitcpio.conf` and move the `block` and `keyboard` hooks before the `autodetect` hook. I forgot when was the initial RAM disk created automatically, maybe just after you install the kernel, but you can always edit the configuration file and run `mkinitcpio -P` at any time.*

Below are some packages or groups you may want to install now:

- `linux419-headers`: I recommend you install this, or you are likely to be told "kernel headers not found" when install some DKMS drivers later.
- `base-devel`: compiler toolchain. Install this if you want to use packages from AUR.
- `trizen` or `yay`: For installing AUR packages.
- `networkmanager`: So you can have internet access in your newly installed OS. WiFi should work out of the box. Install `rp-pppoe` for PPPoE / DSL connection support. See [ArchWiki](https://wiki.archlinux.org/index.php/NetworkManager) if you have a more complex network setup.
- `ntfs-3g`, `dosfstools`, `exfat-utils`, `f2fs-tools` : Deal with various common file systems.
- `btrfs-progs`: Install if you are using btrfs.
- `linux419-zfs`: Install if you are using zfs.
- `manjaro-release`: Offers the `lsb_release` command which should be used by many softwares.
- `broadcom-wl`, `b43-fwcutter`, `ipw2100-fw`, `ipw2200-fw`: Drivers for certain wireless devices. If later you can't connect to WiFi network in the newly installed OS, search your WLAN controller model on the internet, go back to the live environment, chroot again, and decide which packages to install.
- `intel-ucode` or `amd-ucode`, depend on your processor.

Use `mhwd` to auto install graphic drivers needed:

``` bash
# mhwd -a pci free 0300
```

See [Manjaro Wiki](https://wiki.manjaro.org/index.php/Configure_Graphics_Cards) for details of `mhwd`.

### Install Bootloader

Install `grub`. If you have dual/multi booting, also install `os-prober`. Since we are dealing with a UEFI system, `efibootmgr` is also needed. Read the [Manjaro Wiki](https://wiki.manjaro.org/index.php/Restore_the_GRUB_Bootloader) if you are doing BIOS system. `grub-theme-manjaro` can also be installed if you want a prettier grub screen.

Run:

``` sh
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=manjaro --recheck
```

If you are installing the OS to a USB stick, `--removable` parameter is also needed. Then:

``` sh
# update-grub
```

There mighe be and error says "cannot find a GRUB drive for /dev/sdY". If `/dev/sdY` is your installation media, then it's totally fine.

Look into `/boot/grub/grub.cfg` to see if the entry is created correctly. The entry will look like

```
menuentry 'Manjaro Linux (Kernel: 4.19.60-1-MANJARO x64)' ...
```

Other OSes on your machine might not be recognized. Run `update-grub` later in your newly installed OS to fix this.

Run `shutdown now` to shutdown the PC. Remove the installation media, then you should be able to boot into the system.

## After Installation

The first thing to do is connecting to the internet. Run:

``` sh
# systemctl enable NetworkManager.service
# systemctl start NetworkManager.service
```

Then run `nmtui`.

You may want a desktop environment. I'll take i3wm as an example, and show you a way to use the desktop environment without a display manager.

First, install `xorg`, `xorg-init` and `i3`. Put this in your `~/.bash_profile` (or `~/.zprofile` if you are using zsh):

``` bash
if [[ ! ${DISPLAY} && ${XDG_VTNR} == 1 ]]; then
    exec startx
fi
```

which means to start X when login in tty1. See [Gentoo Wiki](https://wiki.gentoo.org/wiki/X_without_Display_Manager) for details.

Put this in your `~/.xinitrc`

``` bash
exec i3
```

Reboot, login, and you will enter the desktop environment. Have fun with it!
