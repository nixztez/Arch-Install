# Arch-Install

## Swedish keyboard
loadkeys sv-latin1

## Internet connection
ping google.se

## Setup wifi if needed
wifi-menu

## Check if we are booted into uefi
efivar -l

## Find the right drive
lsblk

## Wipe drive
gdisk /dev/xxx
x
z
y
y

## Set up partitions
cgdisk /dev/xxx

### Boot partition
1024 MiB - EF00 - boot
### Swap partition
8GiB - 8200 - swap
### Root partition
Rest - 8300 - root

Write and then exit

## Format
### Boot
mkfs.fat -F32 /dev/xxx1
### Swap
mkswap /dev/xxx2
swapon /dev/xxx2
### Root
mkfs.ext4 /dev/xxx3

## Mount
mount /dev/xxx3 /mnt
mkdir /mnt/boot
mount /dev/xxx1 /mnt/boot

## Setup mirrorlist
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.bak
rankmirrors -n 6 /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist

## Setup base system
pacstrap -i /mnt base base-devel

## Generate fstab
genfstab -U -p /mnt >> /mnt/etc/fstab
### Double check it
nano /mnt/etc/fstab

## Chroot into system
arch-chroot /mnt

## Generate locale
nano /etc/locale.gen
locale-gen

## Some stuff
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
echo KEYMAP=sv-latin1 > /etc/vconsole.conf
ln -s /usr/share/zoneinfo/Europe/Stockholm > /etc/localtime
hwclock --systohc --utc
echo iron > /etc/hostname

## Disk trim weekly?
sudo systemctl enable fstrim.timer

## Setup aur
nano /etc/pacman.conf

### Add the following to the bottom
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch

### Update
sudo pacman -Sy
sudo pacman -S yaourt

## Enable multilib
nano /etc/pacman.conf

### Uncomment
[multilib]
Include = /etc/pacman.d/mirrorlist

### Update again
sudo pacman -Sy

## Setup accounts and passwords
### Enter the password for root after this
passwd

### User account
useradd -m -g users -G wheel,storage,power -s /bin/bash mattias
passwd mattias

## Setup sudoers
EDITOR=nano visudo
### Uncomment
%wheel ALL=(ALL) NOPASSWD: ALL
### Add this to the bottom of the file
Defaults rootpw

## Install
sudo pacman -S bash-completion intel-ucode network-manager

## Install boot loader
bootctl install

## Edit boot loader conf
sudo nano /boot/loader/entires/arch.conf
### Add this to the file
title Arch Linux
linux vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img

### From outside file
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/xxx3) rw" >> /boot/loader/entries/arch.conf

## Setup internet
ip link
sudo systemctl enable dhcpcd@modul

sudo pacman -S networkmanager
sudo systemctl enable NetworkManager

## Reboot
exit
umount -R /mnt
sudo reboot

## Install nvidia if needed
sudo pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings linux-headers

## INSTALL DESKTOP ENVIRONMENT AND STUFF
