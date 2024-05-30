# ARCH INSTALL

## Introduction

This covers basic install of arch with luks and lvm, including btrfs

## PREPARATION

Enter iwctl

```
iwctl
```
```
station wlan0 connect <SSID>
```
Write random data
```
dd if=/dev/urandom of=/dev/<NAMEofLSBLK> status=progress bs=8192
```

## PARTITIONING

Inside of gdisk, you can print the table using the `p` command.

To create a new partition use the `n` command. The below table shows 
the disk setup I have for my primary drive

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | +4G         | ef00 |
| 2         | default      | default     | 8309 |

partition 2 will serve as the LUKS and we will create volumes inside of this via lvm + btrfs

## Encryption

Load the encryption modules to be safe.

```
modprobe dm-crypt
modprobe dm-mod
```

Setting up encryption on our luks lvm partiton

```
cryptsetup luksFormat -v -s 512 -h sha512 /dev/<NAME>
```

Enter in your password and **Keep it safe**. There is no "forgot password" here.

Mount the drives:

```
cryptsetup open /dev/<NAME> luks_lvm
```

## Volume setup


Create physical volume

```
pvcreate /dev/mapper/luks_lvm
```

Create volume group
```
vgcreate dropped-arch /dev/mapper/luks_lvm
```

Create a volume for your swap space. A good size for this is your disk space + 2GB.
In my case 66G.
```
lvcreate -L 66G dropped-arch -n swap
```

### Single Disk
If you have a single disk, let's just create two logical volumes:

#### Create two logical volumes
```
lvcreate -L 384G dropped-arch -n root
```

```
lvcreate -l +100%FREE arch -n home
```

## Filesystems

FAT32 on EFI partiton

```
mkfs.fat -F32 /dev/<NAME> 
```

BTRFS on root

```
$ mkfs.btrfs -L root /dev/mapper/dropped--arch-root
```

BTRFS on home if exists

```
$ mkfs.btrfs -L home /dev/mapper/dropped--arch-home
```

Setup swap device

```
$ mkswap /dev/mapper/dropped--arch-swap
```

## Mounting

Mount swap

```
$ swapon /dev/mapper/dropped--arch-swap
$ swapon -a
```

Mount root 

```
$ mount /dev/mapper/dropped--arch-root /mnt
```

Create home and boot

```
$ mkdir -p /mnt/{home,boot}
```

Mount the boot partiton

```
$ mount /dev/<NAME> /mnt/boot
```

Mount the home partition if you have one, otherwise skip this

```
$ mount /dev/mapper/dropped--arch-home /mnt/home
```

With base-devel

```
$ pacstrap -K /mnt base base-devel linux linux-firmware git neovim
```

Load the file table

```
$ genfstab -U -p /mnt > /mnt/etc/fstab
```

chroot into your installation

```
$ arch-chroot /mnt /bin/bash
```

## Configuring

### Text Editor

Install a text editor

```
$ pacman -S neovim
```

```
$ pacman -S nano
```

### Decrypting volumes

Open up mkinitcpio.conf

```
$ nvim /etc/mkinitcpio.conf
```

add `encrypt` and `lvm2` into the hooks

```
HOOKS=(... block encrypt lvm2 filesystems fsck)
```

install lvm2

```
$ pacman -S lvm2
```

### Bootloader

Install grub and efibootmgr

```
$ pacman -S grub efibootmgr
```

Setup grub on efi partition

```
$ grub-install --efi-directory=/boot/efi
```

obtain your lvm partition device UUID

```
lsblk /dev/nvme0n1p3
```

Copy this to your clipboard

```
$ nvim /etc/default/grub
```

Add in the following kernel parameters

```
root=/dev/mapper/arch-root cryptdevice=UUID=<uuid>:luks_lvm
```

### Keyfile

```
$ mkdir /secure
```

Root keyfile
```
$ dd if=/dev/random of=/secure/root_keyfile.bin bs=512 count=8
```

Home keyfile if home partition exists

```
$ dd if=/dev/random of=/secure/home_keyfile.bin bs=512 count=8
```

Change permissions on these

```
$ chmod 000 /secure/*
```

Add to partitions

```
$ cryptsetup luksAddKey /dev/nvme0n1p3 /secure/root_keyfile.bin
# skip below if using single disk
$ cryptsetup luksAddKey /dev/nvme1n1p1 /secure/home_keyfile.bin
```

```
$ nvim /etc/mkinitcpio.conf
```

```
FILES=(/secure/root_keyfile.bin)
```

### Home Partition Crypttab (Skip if single disk)

Get uuid of home partition

```
$ blkid /dev/nvme1n1p1
```

Open up the crypt table.
```
$ nvim /etc/crypttab
```

Add in the following line at the bottom of the table
```
arch-home      UUID=<uuid>    /secure/home_keyfile.bin
```

Reload linux

```
$ mkinitcpio -l linux
```

## Grub

Create grub config

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

## System Configuration

### Timezone

```
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

### NTP

```
$ nvim /etc/systemd/timesyncd.conf
```

Add in the NTP servers

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org 
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

Enable timesyncd

```
# systemctl enable systemd-timesyncd.service
```

### Locale

```
$ nvim /etc/locale.gen
```

uncomment the UTF8 lang you want

```
en_US.UTF-8 UTF-8
```

```
$ locale-gen
```

```
$ nvim /etc/locale.conf
```

```
LANG=en_US.UTF-8
```


### hostname

enter it into your /etc/hostname file

```
$ nvim /etc/hostname
```

or 

```
$ echo "mymachine" > /etc/hostname
```

### Users

First secure the root user by setting a password

```
$ passwd
```

Then install the shell you want

```
$ pacman -S zsh
```

Add a new user as follows

```
$ useradd -m -G wheel -s /bin/zsh user
```

set the password on the user

```
$ passwd user
```

Add the wheel group to sudoers

```
$ EDITOR=nvim visudo
```

```
%wheel ALL=(ALL:ALL) ALL
```

### Network Connectivity

```
$ pacman -S networkmanager
$ systemctl enable NetworkManager
```

### Display Manager

```
$ pacman -S gnome
```

```
$ systemctl enable gdm
```


### Microcode

For AMD

```
$ pacman -S amd-ucode
```

For intel

```
$ pacman -S intel-ucode
```

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```


## Reboot

```
$ exit
$ umount -R /mnt
$ reboot now
```
