# Arch Install Notes
Installing Arch on the Huawei MateBook Pro X with Disk Encryption.

This was done with a new Huawei MateBook Pro X Intel Core i7-8550U 
1.8GHz, 16GB RAM, 512GB SSD. Delivered on 2018/10/08.

This is still very much a WIP, as I go through the install, tweaking things
and improving the doc for consistancy and readabilty.

## Getting Started
I booted into Windows 10 once, and let it go through it's whole setup
only so I could update the BIOS.

Mine shipped with BIOS version 1.17, and I noticed 1.18 was available
on the [Huawei Download Page](ttps://consumer.huawei.com/us/support/pc/matebook-x-pro/) 
so I installed that from Windows 10, before booting from the Arch ISO.

# Booting into the Base Install

## Make an Arch Boot USB Key
Download the [Arch ISO](https://www.archlinux.org/download/)

I burned it to a [UBS Key](https://wiki.archlinux.org/index.php/USB_flash_installation_media#In_macOS) 
on macOS like this:

```
diskutil unmountDisk /dev/disk2
sudo dd if=archlinux-2018.10.01-x86_64.iso of=/dev/rdisk2 bs=1m
```
## Boot from USB

To access the Boot menu, first disable Secure Boot in the BIOS. I've read
that not everyone had to do this, but I couldn't get the boot menu to 
show up until I did this.

Hold down F2 while booting to enter the BIOS.

```
Security Setting -> Secure Boot
    Set to Disable
EXIT
    Save and Exit.

Hold down F12 at boot to enter the boot menu
    Chose EFI USB Device
````

## Make Fonts Readable and Get Online

Make the console font larger so it's readable, we'll set a permenant font 
later:

`# setfont latarcyrheb-sun32`

Following the [Arch Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)

To Get WiFi Working again, chose the wireless network. We'll change
how this is configured later, but this works for now.

`# wifi-menu` 

Check that it works:

`# ping archlinux.org`

Update the system clock

`# timedatectl set-ntp true`

## Partition and Format
I used cfdisk to delete all existing partitions from /dev/nvme0n1, 
removing Windows 10. All operations from here on are done on /dev/nvme0n1.

This install will use full disk encryption, with the exception of the EFI
boot partition. 

Wipe the drive following the [dm-crypt Drive Wipe instructions](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_wipe_on_an_empty_disk_or_partition)
`# cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped`
Verify that the new device exists:

`$ lsblk`

Now wipe it with Zeros, which should be turned into apparent randomness on
disk since this is an encrypted drive.

`# dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress`

This took about 20 minutes to run.

Some debate about effectiveness of this with SSDs, but this can't be
worse than not doing it, also my system was completely new, so I had no
private data.

Close the temporary container

`# cryptsetup close to_be_wiped`

Partition and create a 512MB fat32 partition of type 'EFI System'. 
I used cfdisk to create the parition. I made the rest of the disk type 'Linux 
filesystem'.  Then formatted it:

`# mkfs.fat -F32 /dev/nvme0n1`

## Setup Disk Encryption
I chose the realtively simple LVM on LUKS setup combining instructions from:
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
https://gist.github.com/heppu/6e58b7a174803bc4c43da99642b6094b

There will be a single LUKS2 volume with LVM on top. LVM will then divide
that volume into root, home and swap.

Setup and open the LUKS2 volume

```
# cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
# cryptsetup open /dev/nvme0n1p2 cryptlvm
```

Setup LVM, with swap at least as large as RAM to support hibernate

```
# pvcreate /dev/mapper/cryptlvm
# vgcreate archvg /dev/mapper/cryptlvm
# lvcreate -L 16G archvg -n swap
# lvcreate -L 64G archvg -n root
# lvcreate -l 100%FREE archvg -n home
```

Format the filesystems

```
# mkfs.ext4 /dev/archvg/root
# mkfs.ext4 /dev/archvg/home
# mkswap /dev/archvg/swap
```

Mount the partitions

```
# mount /dev/archvg/root /mnt
# mkdir /mnt/home
# mount /dev/archvg/home /mnt/home
# swapon /dev/archvg/swap
```

Mount the boot/ESP volume

```
# mkdir /mnt/boot
# mount /dev/nvme0n1p1 /mnt/boot
```

Install the base system

`# pacstrap /mnt base base-devel`

Generate the fstab

`# genfstab -U /mnt >> /mnt/etc/fstab`

## Basic Configs for the New Install
This is mostly straight from the [Arch Wiki Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide)

Chroot to the new arch install

`# arch-chroot /mnt`

Set the timezone and hwclock

```
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
# hwclokc --systohc
```

Setup Locales, uncomment any needed in /etc/locale.gen and generate
them

```
# locale-gen`
# echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

Set a hostname

`# echo archbook >> /etc/hostname`

Install the systemd-boot boot loader. I'm using this because it's simple
to setup and configure.

`# bootctl --path=/boot install`

Edit /etc/mkinitcpio.conf

```
MODULES=(ext4)
...
...
...
HOOKS=(base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck)
```

Install Intel Microcode Updates, this will install an initrd image that 
we add to our boot loader config.

`# pacman -S intel-ucode`

Create /boot/loader/entries/arch.conf

```
title   Arch Linux
linux   /vmlinuz-linux
initrd /intel-ucode.img
initrd  /initramfs-linux.img
options cryptdevice=UUID=<YOUR-PARTITION-UUID>:lvm:allow-discards resume=/dev/mapper/archvg-swap root=/dev/mapper/archvg-root rw quiet
```

Edit /boot/loader/loader.conf

```
timeout 0
default arch
editor 0
```


Regnerate initramfs

`# mkinitcpio -p linux `

Install some packages so wifi-menu works later

`# pacman -S dialog wpa_supplicant`

Set the root password

`# passwd`

Reboot

```
# exit
# reboot
```

# Post Install Config

Get wifi back online after first boot. Will change how this is configured
later.

`# wifi-menu`


## Get X11 Working

Install X11 and the i3 window manager
```
# pacman -S i3 nvidia xorg-server xorg-font-util xorg-fonts-75dpi \
# xorg-fonts-100dpi xorg-mkfontdir xorg-mkfontscale xorg-xdpyinfo \
# xorg-xrandr xorg-xset bumblebee bbswitch
```

Enable Bumblebee with bbswitch for Nvidia / Intel switching
```
# systemctl enable bumblebeed.service
# gpasswd -a $USER bumblebee
```

Install and setup etckeeper
```
# pacman -S etckeeper
# cd /etc
# etckeeper init
# etckeeper commit
```

Enable TLP for powersaving
```
# pacman -S tlp tlp-rdw
# systemctl enable tlp.service
# systemctl enable tlp-sleep.service
# systemctl mask systemd-rfkill.service
# systemctl mask systemd-rfkill.socket
# systemctl enable NetworkManager-dispatcher.service
```

Get X11 running
```
# pacman -S lightdm
# pacman -S lightdm-gtk-greeter
# pacman -S termite
# pacman -S firefox
# pacman -S mesa
# pacman -S xf86-video-intel
```

Get the Network to come up automatically
```
# nmcli device wifi connect <Network> password <password>
# pacman -S network-manager-applet
```

Install git and ssh so I can update this README from the MateBook
```
pacman -S git
pacman -S openssh
```

## Neovim and Shell Setup

```
pacman -S nvim
pacman -S neovim
pacman -S zsh
pacman -S python3
```

Install my homshick setup, bootstrap of oh-my-zsh needs some automation.

Install YaY to build packages from Arch AUR

```
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

Install things needed for my custom i3 setup

```
$ yay bumblebee-status
# pacman -S dmenu
# pacman -S feh
# pacman -S pacman-contrib
```

A few more things to make X happy.
```
# pacman -S xss-lock compton redshift
```
Redshift configs come from https://github.com/kelp/redshift 

Install Nerd Fonts Complete, these are used by my terminal, required for my nvim and shell config

`yay nerd-fonts-complete`

Fix sound

`# pacman -S alsa-tools`
Follow instructions at the bottom of this:
https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html

Sort the pacman mirrors
https://wiki.archlinux.org/index.php/mirrors#Sorting_mirrors

Update systemd-boot when systemd is upgraded. From:
https://wiki.archlinux.org/index.php/Systemd-boot#Automatic_update

`$ yay systemd-boot-pacman-hook`

Install gnome-keyring to manage ssh keys

`# pacman -S gnome-keyring`

Set it up with the PAM method for console. LightDM should handle it for X11
https://wiki.archlinux.org/index.php/GNOME/Keyring#PAM_method

Screen locking with Google's
`$ yay xsecurelock`

Install xbacklight
`# pacman -S xorg-xbacklight`

Install xscreensaver
`# pacman -S xscreensaver`

Set a readable console font:

`# pacman -S terminus-font`

Create `/etc/vconso-e.conf` with contents:

`FONT=ter-132n`

Then add 'consolefont' to the HOOKS section of /etc/mkinitcpio.conf and run:

`# mkinitcpio -p linux`

From: https://wiki.archlinux.org/index.php/Laptop#Touchpad
Hibernate on low battery to '/etc/udev/rules.d/99-lowbat.rules'
```# Suspend the system when battery level drops to 5% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-5]", RUN+="/usr/bin/systemctl hibernate"

```
Setup time sync
https://wiki.archlinux.org/index.php/Systemd-timesyncd
Edit /etc/timesyncd.conf uncomment these lines:
```
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.us.pool.ntp.org
```
Then start it:

`# timedatectl set-ntp true `

Enable macOS like touch pad scrolling, create:

`/etc/X11/xorg.conf.d/30-touchpad.conf`
```
Section "InputClass"
    Identifier "touchpad"
    Driver "libinput"
    MatchIsTouchpad "on"
    Option "NaturalScrolling" "true"
EndSection
```

Disable the Touch Screen (why would anyone want that?!)
Create: `/etc/X11/xorg.conf.d`
With contents:
```
Section "InputClass"
    Identifier         "Touchscreen catchall"
    MatchIsTouchscreen "on"

    Option "Ignore" "on"
EndSection
```

TODO
https://wiki.archlinux.org/index.php/Microcode#systemd-boot
https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html
https://bentley.link/secureboot/
https://wiki.archlinux.org/index.php/Secure_Boot
https://wiki.archlinux.org/index.php/HiDPI
* Setup Power management
* Setup Sound
* Fix sound
* Make screen brightness and volume keys work
* Fix firefox screentearing
* Setup slick-greeter for lightdm
* Setup background that fits with screen resolution
* Setup boot splash screen with arch logo
