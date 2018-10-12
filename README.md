# arch
Documentation and things related to my arch setup

# Arch Install Notes
Installing Arch on the Huawei MateBook Pro X with Disk Encryption.

This was done with a new Huawei MateBook Pro X Intel Core i7-8550U 
1.8GHz, 16GB RAM, 512GB SSD. 

## Detailed Notes
I booted into Windows 10 once, and let it go through it's whole setup.

Mine shipped with BIOS version 1.17, and I noticed 1.18 was available
on the [Huawei Download Page](ttps://consumer.huawei.com/us/support/pc/matebook-x-pro/)
so I installd that from Windows 10, before booting from the Arch ISO.

# Base Install

## Make an Arch Boot USB Key
Download the [Arch ISO](https://www.archlinux.org/download/)
Burn it to a [UBS Key](https://wiki.archlinux.org/index.php/USB_flash_installation_media#In_macOS)On macOS I did this:
```
diskutil unmountDisk /dev/disk2
sudo dd if=archlinux-2018.10.01-x86_64.iso of=/dev/rdisk2 bs=1m
```

## Boot from USB

To access the Boot menu, first disable Secure Boot in the BIOS. I've read
that not everyone had to do this, but I couldn't get the boot menu to 
show up until I did this.

Hold down F2 while booting to enter the BIOS.
Security Setting -> Secure Boot
    Set to Disable
EXIT
    Save and Exit.

Hold down F12 at boot to enter the boot menu
    Chose EFI USB Device

Make the console font larger:
`# setfont latarcyrheb-sun32`

Following the [Arch Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)

Get WiFi Working, chose the wireless network
`# wifi-menu` 

Check that it works:
`# ping archlinux.org`

Update the system clock
`# timedatectl set-ntp true`

I used cfdisk to delete all existing partitions from /dev/nvme0n1, 
removing Windows 10.

This is going to use full disk encryption, so we'll set that up now.

Wipe the drive following the [dm-crypt Drive Wipe instructions](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_wipe_on_an_empty_disk_or_partition)
`# cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped`
Verify that the new device exists:
`$ lsblk`

Now wipe it with Zeros, which should be turned into randomness on
disk since this is an encrypted drive.

`# dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress`
This took about 20 minutes to run.

Some debate about effectiveness of this with SSDs, but this can't be
worse than not doing it.

Close the temporary container
`# cryptsetup close to_be_wiped`

Partition and create a 512MB fat 32 EFI parition. I used cfdisk to
create the parition. I made the rest of the disk a Linux partition.
Then formatted it:
`# mkfs.fat -F32 /dev/nvme0n1`

Combining instructions from:
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
https://gist.github.com/heppu/6e58b7a174803bc4c43da99642b6094b

Setup and open the LUKS2 volume
```
# cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
# cryptsetup open /dev/nvme0n1p2 cryptlvm
```
Setup LVM
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

Install the systemd-boot boot loader
`# bootctl --path=/boot install`

Edit /etc/mkinitcpio.conf

```
MODULES=(ext4)
...
...
...
HOOKS=(base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck)
```


Create /boot/loader/entries/arch.conf

```
title   Arch Linux
linux   /vmlinuz-linux
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

Install some packages to wifi-menu works later
`# pacman -S dialog wpa_supplicant`

Set the root password
`# passwd`

Reboot
```
# exit
# reboot
```
# Post Install Config

On first boot, wifi didn't work, so I got it online temporarily
with:
`# wifi-menu`

## Intel Microcode Updates
`# pacman -S intel-ucode`


TODO
https://wiki.archlinux.org/index.php/Network_configuration#Network_managers
https://wiki.archlinux.org/index.php/Microcode#systemd-boot
https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html
https://bentley.link/secureboot/
https://wiki.archlinux.org/index.php/Secure_Boot
https://wiki.archlinux.org/index.php/Systemd-boot#Automatic_update
* Setup X11
* Setup Power management
* Setup Sound
* Install binary nvidia drivers
* Increase the size of the console font
* Make WiFi work after reboot and auto-connect to known networks.
