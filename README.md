# Arch-Install

## Load appropriate keyboard layout
An example of how to load the swedish layout:
```
loadkeys sv-latin1
```

## Check if we are booted into uefi
The following command should list a bunch of variables if booted into uefi, this is important as this is a uefi install. If you are not, use Google.
```
efivar -l
```

## Internet connection
If you are connected using ethernet it should already be working, if you are on a laptop which doesn't have an ethernet port you are required to get an adapter or use a driverless usb wireless adapter. If you wish to load your required wireless drivers, use Google. 

When you are ready to connect to a wirless network, use the command below.
```
wifi-menu
```

Check your internet connection by pinging a website, in this case google.
```
ping google.se
```

## Partitioning
List all drives and partitions. If you only have one drive it will most likely be /dev/sda, unless it is an nvme drive. Take note of which drive you want to work with by checking existing partitions and the size of the drive.
```
lsblk
```

If you wish to wipe the entire drive you can do that with the following command where /dev/xxx is the drive you decided to use.
```
gdisk /dev/xxx
x
z
y
y
```

Next up use cgdisk on your chosen drive to setup partitions. Here you can also choose to delete certain partitions to create new ones if you wish to, this is useful if you are going to dual boot.
```
cgdisk /dev/xxx
```

### Boot
The first partition we are going to create is the boot partition. If you want some extra space in the boot partition just in case you need it, make it 1024MiB. If you have a very limited amount of space, make it 550MiB as recommended by the arch wiki. The hex code you want to use is EF00. This partition will also be automatically detected by macs in the boot menu.

### Swap
Next you have to decide if you want to create a swap partition or a swap file. According to the wiki, the swap file do not have any performance overhead compared to a partition but are much easier to resize as needed. Hence the swap file is recommended if you have a limited amount of space or you are not sure of how big you want to create it. Please refer to the wiki article below if you have trouble choosing size of the partition. The hex code of swap is 8200.

### Root
Create a root partition with remaining space and give it the default hex code of 8300. It is also quite popular to make the home folder  its own partition - that way it's really easy to reinstall the system without losing all the data in /home. I'll however leave that as an excercise for the reader.

### Finishing up and more
Before writing the changes, make sure you haven't deleted any partition you actually want to keep. Write the changes and then exit cgdisk.

If you wish to read more about partitioning and best practices read the arch wiki [here](https://wiki.archlinux.org/index.php/partitioning).

## Formatting
Note that the specified partition numbers might not actually match yours, so make make sure to use **lsblk** to get the correct partitions.

### Boot
```
mkfs.fat -F32 /dev/xxx1
```

### Swap
Only do this step if you decided to create a swap partition. We will get to the swap file soon.
```
mkswap /dev/xxx2  
swapon /dev/xxx2
```

### Root
```
mkfs.ext4 /dev/xxx3
```

## Mounting the partitions
Mount your new partitions to **/mnt**. You should mount the root partition to **/mnt** and the boot partition to **/mnt/boot**.
```
mount /dev/xxx3 /mnt  
mkdir /mnt/boot  
mount /dev/xxx1 /mnt/boot
```

## Setup mirrorlist
Please take the time to rank the mirrors as these will be copied onto your live system as well by pacstrap. 
```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak  
sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.bak  
rankmirrors -n 6 /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

## Setup base system
It is not absolutely necessary to install the **base-devel** package group for a minimal install, but it is needed to use the **Arch User Repository** (AUR).
```
pacstrap -i /mnt base base-devel
```

## Generate fstab
This will generate the fstab file which is used to define where partitions should be mounted into the filesystem.
```
genfstab -U /mnt >> /mnt/etc/fstab
```

## Chroot into system
To change root (chroot) into the the system, the following command needs to be executed. This is called a **chroot jail** as it is no longer possible to access anything outside of /mnt/. Hence, / inside the jail will be the root partition as it was mounted there earlier. This is really useful if a system is unable to boot as this gives access to the filesystem.
```
arch-chroot /mnt
```

## Install vim
You may want to use nano if you haven't used vim before, nonetheless it is great to learn.
```
pacman -S vim
```

## Creating swap file
In this example the swapfile will be 512MiB (Mebibyte). Replace **512** with an appropriate size for the system.
```
fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```
To modify the swapfile after creation, please refer to the following [arch wiki](https://wiki.archlinux.org/index.php/Swap#Swap_file).

Since there currently is no entry for the swap file in **/etc/fstab**, it now has to be added.
```
vim /mnt/etc/fstab
```

The entry should look as following.
```
/swapfile none swap defaults 0 0
```

## fstab
Open **/mnt/etc/fstab** again and check for errors such as missing entries. At this point there should be one entry for boot, swap partition / swap file and the root partition.

## Generate locale
Locales are used for rendering text, correctly displaying regional monetary values, time and date formats, alphabetic idiosyncrasies, and other locale-specific standards. Uncomment the locale of choice in the following file.
```
vim /etc/locale.gen
```

Generate the locale(s)
```
locale-gen
```

## Some stuff
```
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo KEYMAP=sv-latin1 > /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
hwclock --systohc --utc
echo myhostname > /etc/hostname
```

## Weekly disk trim
To enable weekly disk trim execute the following command.
```
sudo systemctl enable fstrim.timer
```

## Enable multilib
```
vim /etc/pacman.conf
```

### Uncomment
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

## Setup aur
Install the "yay" AUR helper https://github.com/Jguer/yay
```
sudo pacman -Sy
```

## Setup accounts and passwords
### Enter the password for root after this
```
passwd
```
### User account
```
useradd -m -g users -G wheel,storage,power -s /bin/bash myusername  
passwd myusername
```

## Setup sudoers
```
EDITOR=vim visudo
```
### Uncomment
```
%wheel ALL=(ALL) ALL
```
### Add this to the bottom of the file
```
Defaults rootpw
```

## Install
```
sudo pacman -S bash-completion intel-ucode
```

## Install boot loader
```
bootctl install
```

## Edit boot loader conf
```
sudo vim /boot/loader/entires/arch.conf
```
### Add this to the file
```
title Arch Linux
linux vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
```

### From outside file
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/xxxx) rw" >> /boot/loader/entries/arch.conf

## Setup internet
```
sudo pacman -S networkmanager  
sudo systemctl enable NetworkManager
```

## Reboot
```
exit  
umount -R /mnt  
reboot
```

## INSTALL DISPLAY SERVER, DESKTOP ENVIRONMENT AND STUFF
