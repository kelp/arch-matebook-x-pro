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
$ diskutil unmountDisk /dev/disk2
$ sudo dd if=archlinux-2018.10.01-x86_64.iso of=/dev/rdisk2 bs=1m
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

Most of this next part is from the [Arch Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)

To Get WiFi working chose the wireless network. We'll change
how this is configured later, but this works for now.

`# wifi-menu` 

Check that it works, it may take a few seconds for netowrking to come up.

`# ping archlinux.org`

Update the system clock

`# timedatectl set-ntp true`

## Partition and Format

First we wipe the drive, this will destroy any existing data and partitions,
overwriting all data with randomness.

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

Close the temporary container

`# cryptsetup close to_be_wiped`

Partition and create a 512MB fat32 partition of type 'EFI System'. 
I used cfdisk to create the parition. I made a 220GB partition 
with type 'Linux filesystem' for Arch. The rest will be reserved for future
OS installs.

Format the EFI/boot volume:

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
# hwclock --systohc
```

Setup Locales, uncomment any needed in /etc/locale.gen and generate
them

```
# locale-gen
# echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

Set a hostname

`# echo archbook >> /etc/hostname`

## Make the system bootable

Install Intel Microcode Updates, this will install an initrd image that 
we add to our boot loader config.

`# pacman -S intel-ucode`

Next setup rEFInd it's themable and looks much nicer than grub or 
systemd-boot

`# pacman -S refind-efi parted sbsigntools imagemagick`
`# refind-install`

Add a menu entry for Arch, and some theme configs
not strictly necessary, but it makes the nice icons
show up and gives us some submenus. Goes into `/boot/EFI/refind/refind.conf`

The UUID below must be the UUID returned from running:

`blkid /dev/nvme0n1p2`

```
use_graphics_for linux
...
showtools shell, memtest, gdisk, mok_tool, about, shutdown, reboot, firmware, fwupdate
...
scanfor manual
...
menuentry "Arch Linux" {
    icon     /EFI/refind/icons/os_arch.png
    volume   "Arch Linux"
    loader   /vmlinuz-linux
    initrd   /initramfs-linux.img
    options  "cryptdevice=UUID=<YOUR-PARTITION-UUID>:lvm:allow-discards resume=/dev/mapper/archvg-swap root=/dev/mapper/archvg-root initrd=/intel-ucode.img rw quiet splash"
    submenuentry "Boot using fallback initramfs" {
        initrd /initramfs-linux-fallback.img
    }
    submenuentry "Boot to terminal" {
        add_options "systemd.unit=multi-user.target"
    }
}
```

Edit /etc/mkinitcpio.conf and add encrypt, lvm2, and resume support, the 
postion in the HOOKS line matters:

```
MODULES=(ext4)
...
...
...
HOOKS=(base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck)
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

Install and setup etckeeper
```
# pacman -S etckeeper
# cd /etc
# etckeeper init
# etckeeper commit
```

## Create a regular user
```
# useradd -G wheel -m kelp
# passwd kelp
```

Then I log out and switch to that user.

## Setup power savings

Enable TLP for powersaving
```
# pacman -S tlp tlp-rdw
# systemctl enable tlp.service
# systemctl enable tlp-sleep.service
# systemctl mask systemd-rfkill.service
# systemctl mask systemd-rfkill.socket
# systemctl enable NetworkManager-dispatcher.service
```

Install ethtool lsb-release and smartmontools at the suggestion of tlp-stat

`# pacman -S ethtool lsb-release smartmontools`

Get the Network to come up automatically
```
# systemctl start NetworkManager.service
$ nmcli device wifi connect <Network> password <password>
```

Install ssh so I can update this README from the MateBook
```
# pacman -S openssh
```

## Get X11 Working

Install X11, i3wm, lightdm, and a few other nice things
```
# pacman -S i3 nvidia xorg-server xorg-font-util xorg-fonts-75dpi 
# pacman -S xorg-fonts-100dpi xorg-mkfontdir xorg-mkfontscale xorg-xdpyinfo 
# pacman -S xorg-xrandr xorg-xset bumblebee bbswitch
# pacman -S lightdm lightdm-gtk-greeter termite
# pacman -S firefox mesa xf86-video-intel
# pacman -S network-manager-applet
# pacman -S xbindkeys xorg-xmodmap xorg-xrdb
```

Enable Bumblebee with bbswitch for Nvidia / Intel switching
```
# systemctl enable bumblebeed.service
# gpasswd -a $USER bumblebee
```

## Neovim and Shell Setup

```
# pacman -S neovim zsh python3 python-neovim
```
## Setup my dot files, includes x11, nvim and zsh configs
Install my homshick setup https://github.com/kelp/dotfiles

Install YaY to build packages from Arch AUR

```
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

Install things needed for my custom i3 setup. For bumblebee-status
I use the git version because it has a bug fix I sent upstream.

```
$ yay bumblebee-status
# pacman -S dmenu feh pacman-contrib python-dbus
```

A few more things to make X happy.

`# pacman -S xss-lock compton redshift`

Redshift configs come from https://github.com/kelp/redshift 

Install Nerd Fonts Complete, these are used by my terminal, required for my 
nvim and shell config

`$ yay nerd-fonts-complete`

Screen locking with Google's

`$ yay xsecurelock`
Depends on my .xprofile, which I swiched to use xsecurelock

Install xscreensaver

`# pacman -S xscreensaver`

## Start LightDM
At this point X11 should work, and we can use it, though we'll still be tweaking
a bunch of things.
`# systemctl start lightdm`

Now login to X11!

## Make Pacman faster

Sort the pacman mirrors by speed
https://wiki.archlinux.org/index.php/mirrors#Sorting_mirrors

Install gnome-keyring to manage ssh keys

`# pacman -S gnome-keyring`

Set it up with the PAM method for console. LightDM should handle it for X11
https://wiki.archlinux.org/index.php/GNOME/Keyring#PAM_method


## Setup the Linux Console

Set a readable console font:

`# pacman -S terminus-font`

Create `/etc/vconsole.conf` with contents:

`FONT=ter-132n`

Then add 'i915' to MODULES and 'consolefont' to the HOOKS section of 
/etc/mkinitcpio.conf:
```
MODULES=(i915 ext4)
...
HOOKS=(base udev autodetect consolefont modconf block lvm2 resume filesystems keyboard fsck)
```
And run: 
`# mkinitcpio -p linux`

If i915 isn't added here, when you reboot, you'll see the new conole font
briefly and then it will reset to the tiny ones.

At this point I rebooted to test all this out.

## Configure the Trackpad

## Configure Synaptics
Info from: https://williambharding.com/blog/linux-to-macbook/linux-with-a-macbook-touchpad-feel-pt-2/
I'm using Synamptics based on the recommendations from the above Blog Post.
I was finding the trackpad behavior pretty annoying, given I spent most of my 
days on a Mac and have been using Macs for over a decade. Inspite of Synaptis 
limited support going forward, it seems to work better so far.

Here is how I configured it.

```
# pacman -S xf86-input-synaptics
```

Then I created /etc/X11/xorg.conf.d/30-synaptics.conf with these contents:
```
Section "InputClass"
        Identifier "touchpad catchall"
        Driver "synaptics"
        MatchIsTouchpad "on"
        # Enabling tap-to-click is a perilous choice that begets needing to set up palm detection/ignoring. Since I am fine clicking my touchpad, I sidestep the issue by disabling tapping. 
        Option "TapButton1" "0"
        Option "TapButton2" "0"
        Option "TapButton3" "0"
	# Using negative values for ScrollDelta implements natural scroll, a la Macbook default. 
        Option "VertScrollDelta" "-80"
	Option "HorizScrollDelta" "-80"
        # https://wiki.archlinux.org/index.php/Touchpad_Synaptics has a very buried note about this option
	# tl;dr this defines right button to be rightmost 7% and bottommost 5%
	Option "SoftButtonAreas" "93% 0 95% 0 0 0 0 0"  
        MatchDevicePath "/dev/input/event*"
EndSection
```

## Hibernate on Low Battery

From: https://wiki.archlinux.org/index.php/Laptop#Touchpad
Hibernate on low battery to '/etc/udev/rules.d/99-lowbat.rules'
```# Suspend the system when battery level drops to 5% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-5]", RUN+="/usr/bin/systemctl hibernate"

```

## Time Sync

Setup time sync
https://wiki.archlinux.org/index.php/Systemd-timesyncd
Edit /etc/systemd/timesyncd.conf uncomment these lines:
```
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.us.pool.ntp.org
```
Then start it:

`# timedatectl set-ntp true `


## Setup lightdm and slick-greeter

Install lightdm-slick-greeter and lightdm-settings

```
$ yay lightdm-slick-greeter 
$ yay lightdm-settings
```

LightDM config '/etc/lighdm/lightdm.conf':
```
[Seat:*]
greeter-session=lightdm-slick-greeter
```

Slick Greeter Config '/etc/lightdm/slick-greeter.conf:
```
[Greeter]
background = /usr/share/slick-greeter/arch-2560x1600.png
show-hostname = true
show-power = true
show-click = true
show-quit = true
show-a11y = false
draw-grid = false
show-keyboard= false
draw-user-backgrounds = false
```

Make the 'Dynamic User' stop showing up on the login screen, by updating
/etc/lightdm/users.conf to exclude users with /sbin/nologin as their shell

`hidden-shells=/bin/false /usr/bin/nologin /sbin/nologin`

Setup Plymouth
Following instructions from: https://wiki.archlinux.org/index.php/plymouth

```
$ yay plymouth 
$ yay plymouth-theme-arch-breeze-git
```

Set the theme in /etc/plymouth/plymouthd.conf

```
[Daemon]
Theme=arch-breeze
ShowDelay=5
DeviceTimeout=5
```

Disable lightdm systemd unit and enable the lightdm-plymouth unit.

```
# systemctl disable lightdm.service
# systemctl enable lightdm-plymouth.service
```

Install ttf-dejavu fonts, Plymouth needs this, mkinitcpio will fail without it

`$ pacman -S ttf-dejavu`

Add Plymouth requirements to /etc/mkinitcpio.conf. The plymouth-encrypt
is required when running disk encryption. I couldn't boot the first time through
because I missed that.

```
HOOKS=(base udev plymouth consolefont autodetect modconf block plymouth-encrypt lvm2 resume filesystems keyboard fsck)
```

Run mkinitcpio

`# mkinitcpio -p linux`

## Make Sound Work

This took some experimenting. At first all the speakers did't work
and it sounded horrible.

First I installed these packages

`# pacman -S alsa-utils pulseaudio pulseaudio-alsa alsa-tools pavucontrol pulsemixer`


Follow instructions at the bottom of this

https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html

But I also referenced these:
https://imgur.com/a/N1xsCVZ

https://www.reddit.com/r/MatebookXPro/comments/8z4pv7/fix_for_the_2_out_of_4_speakers_issue_on_linux/

I ended up having to run `hdajackretask` as root from a terminal to get it
to apply the configs properly. It creates `hda-jack-retask.conf`
and `/usr/lib/firmware/hda-jack-retask.fw`. I also had to set Connectivity
to Internal for both pins.  Once I did that, and rebooted, I was able
to run `pavucontrol` and chose 'Analog Surround 4.0 Output' from the
Configuration tab. And then on the Output Devices tab, I could unlock
channels and verify that all 4 speakers were working, and control volume
for each.

## Make the Volume and Screen Brighness Buttons Work
Instructions generally came from:
https://wiki.archlinux.org/index.php/Xbindkeys

To set the backlight we need xbacklight

`# pacman -S xorg-xbacklight`

We'll need xbindkeys
```
# pacman -S xbindkeys
$ xbindkeys -d > ~/.xbindkeysrc
```
Add these to .xbindkeysrc:
```
# Increase volume
"pactl set-sink-volume @DEFAULT_SINK@ +1000"
   XF86AudioRaiseVolume

# Decrease volume
"pactl set-sink-volume @DEFAULT_SINK@ -1000"
   XF86AudioLowerVolume

# Mute volume
"pactl set-sink-mute @DEFAULT_SINK@ toggle"
   XF86AudioMute

# Increase backlight
"xbacklight -inc 10"
   XF86MonBrightnessUp

# Decrease backlight
"xbacklight -dec 10"
   XF86MonBrightnessDown
```

Make backlight changes work, create `/etc/X11/xorg.conf.d/20-intel.conf`
with contents:
```
Section "Device"
    Identifier  "Card0"
    Driver      "intel"
    Option      "Backlight"  "intel_backlight"
EndSection
```
And then restart X or reboot.


## Install a Decent Theme for refind
`$ yay refind-theme-regular`

I also decided I didn't like the bright white background of that theme, so I
edited the theme config `/boot/EFI/refind/refind-theme-regular/theme.conf` 
and swapped in an all black 32x32 pixel png for the background. I got the 
image from here: https://dummyimage.com/32x32/000/000000

Add to the bottom of `/boot/EFI/refind/refind.conf` to add a nice theme.

```
include refind-theme-regular/theme.conf
```

Get the pacman hook setup to install refind on the EFI partition on upgrade
https://wiki.archlinux.org/index.php/REFInd#Pacman_hook


Other packages I install:
```
bc
unzip
mlocate
```

TODO
* https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html
* https://bentley.link/secureboot/
* https://wiki.archlinux.org/index.php/Secure_Boot
* https://wiki.archlinux.org/index.php/HiDPI
* Make plymouth and slickgreeter have the same background for a consistent 
  boot experience
* Setup Dunst or similar to show notifcations on volume and brightness change
    https://wiki.archlinux.org/index.php/Dunst
