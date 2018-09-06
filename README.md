# Arch-Install

**Prerequisites:** A drive that can be wiped, or if dual booting, enough with remaining space to create partitions for Arch.

**Always** check if the correct partition and drive is used when executing a command as the examples throughout this tutorial might not be the correct for you.

Throughout this tutorial vim is used, and that is interchangeable with nano if you don't know how to use vim. If you accidentally open vim, close it with **:q!**.

## Load appropriate keyboard layout
An example of how to load the swedish layout:
```
loadkeys sv-latin1
```

## Check if booted into uefi mode
The following command should list a bunch of variables if booted into uefi, this is important as this is a uefi install. If the variables is not listed, use Google.
```
efivar -l
```

## Connecting to the internet
All connections using ethernet should already be working, and on laptops which doesn't have an ethernet port, it would be really convenient to use an ethernet adapter or use a driverless usb wireless adapter. Please refer to google if attempting to use wireless drivers during install. To connect to a wireless network, use the **wifi-menu** command.

Verify internet connection by pinging a website, in this case Google.
```
ping google.se
```

## Partitioning
List all drives and partitions with **lsblk**. Take note of the drive the installation is going to take place on by checking existing partitions and the size of the drive.
```
lsblk
```

If the entire drive is going to be used solely for arch, wipe it with the following commands where **/dev/xxx** is the drive.
```
gdisk /dev/xxx
x
z
y
y
```

Next up, execute **cgdisk /dev/xxx/** on the chosen drive and create the following partitions on top of the existing ones if dual booting.

### Boot
The first needed partition is the boot partition. Arch wiki recommends a size of 550MiB for efi systems, but a bigger may be needed if playing around with multiple kernels. The hex code to create it with is EF00. This partition will also be detected by macs boot menu.

### Swap
Now there is two options, and the decision is between using a swap partition or a swap file. According to the wiki, the swap file do not have any performance overhead compared to a partition but are much easier to resize as needed. Hence, the swap file is recommended if there is a limited amount of space or if there is a high chance that it will be changed. Please refer to the wiki article below for recommendations regarding the size of the partition. The hex code of swap is 8200.

### Root
Create a root partition with remaining space and give it the default hex code of 8300. It is also quite popular to make the home folder its own partition - that way it's really easy to reinstall the system without losing all the data in /home. I'll however leave that as an excercise for the reader.

### Finishing up and more
Before writing the changes, make sure all of the wanted partitions are still there. Write the changes and then exit cgdisk.

If you wish to read more about partitioning and best practices read the arch wiki [here](https://wiki.archlinux.org/index.php/partitioning).

## Formatting
Note that the specified partition numbers might not actually match, so make make sure to use **lsblk** to get the correct partitions.

### Boot
```
mkfs.fat -F32 /dev/xxx1
```

### Swap
Only do this step if a swap partition was created earlier.
```
mkswap /dev/xxx2
swapon /dev/xxx2
```

### Root
```
mkfs.ext4 /dev/xxx3
```

## Mounting the partitions
Mount the root partition to **/mnt** and the boot partition to **/mnt/boot**.
```
mount /dev/xxx3 /mnt
mkdir /mnt/boot
mount /dev/xxx1 /mnt/boot
```

## Setup mirrorlist
Please take the time to rank the mirrors as these will be copied onto the live system by pacstrap. 
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

## Installing vim
You may want to use nano if you haven't used vim before, nonetheless it is great to learn. If planning on using **gvim**, install that package instead as it conflicts with **vim**. This is not a problem since **gvim** also provides the terminal variant.
```
pacman -S vim
```

## Creating a swap file
In this example the swapfile will be 512MiB (Mebibyte). Replace **512** with an appropriate size for the system.
```
fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```
To modify the swapfile after creation, please refer to the following [arch wiki](https://wiki.archlinux.org/index.php/Swap#Swap_file).

Since there currently is no entry for the swap file in fstab, edit it with **vim /etc/fstab**. Add an entry that looks as following.
```
/swapfile none swap defaults 0 0
```

## fstab
Open **/mnt/etc/fstab** again and check for errors such as missing entries. At this point there should be one entry for each of the following partitions.

boot\
swap partition/swap file\
root partition

## Configuring locale
Locales are used for rendering text, correctly displaying regional monetary values, time and date formats, alphabetic idiosyncrasies, and other locale-specific standards. Uncomment the locale of choice in the following file.
```
vim /etc/locale.gen
```

Generate the locale(s).
```
locale-gen
```

Set the locale by typing **LANG=en_US.UTF-8** where **en_us.UTF-8** is the chosen locale into **/etc/locale.conf**.
```
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

## Keymap
Set the keymap by typing **KEYMAP=sv-latin1** where **sv-latin1** is the chosen locale into **/etc/vconsole.conf**
```
echo KEYMAP=sv-latin1 > /etc/vconsole.conf
```

## Setting timezone
Use tab complete in **/usr/share/zoneinfo/** to see available choices and symlink it to **/etc/localtime**.
```
ln -sf /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
hwclock --systohc --utc
```

## Setting hostname
```
echo myhostname > /etc/hostname
```

## Weekly disk trim
To enable weekly disk trim execute the following command.
```
sudo systemctl enable fstrim.timer
```

## Enabling multilib
Multilib contains 32-bit software and libraries that can be used to run and build 32-bit applications on 64-bit installs. To enable it,
edit **/etc/pacman.conf** and uncomment the following lines. 
```
Color

[multilib]
Include = /etc/pacman.d/mirrorlist
```
After **/etc/pacman.conf** has been saved, update the package manager with **pacman -Sy**.

## Setting up accounts and passwords
To change the password for root, execute **passwd**.

To add a regular user to the system, execute the following commands. There are many ways to give a user full root access, in this scenario the user is added to the wheel group. Note that the group itself doesn't give the user root access, that has to be explicitly told in the sudoers file. An alternative way is to only add that specific user to the sudoers file.
```
useradd -m -G wheel myusername
passwd myusername
```
The **-m** option is short for **--create-home** and is very self explanatory. The **-G** is short for **--groups** and is a list of supplementary groups of the new user. **-G** is not the same as **-g** which only specifies the primary group of the user that will be given to files created by the user. The default value of **-g** is the same as the username.

## Setting up sudoers
Never edit the sudoers file by opening it with a regular text editor like **vim /etc/sudoers**. The problem with this is that it never checks the syntax of the file before writing it, and that can result in being locked out of sudo. Instead, always use the visudo program as that will not allow a faulty configuration to be saved. Open the visudo file with **EDITOR=vim visudo** and uncomment the following line.
```
%wheel ALL=(ALL) ALL
```

## Installing a few packages

The next bit will be dependent on the processor used in the system. In the following example a package called **intel-ucode** is installed, as well as two other useful packages. Replace **intel-ucode** with **amd-ucode** if the processor used is an amd.
```
pacman -S bash-completion git linux-headers intel-ucode
```

## Installing the boot loader
Next, install the systemd-boot by executing the following command.
```
bootctl install
```

Edit **/boot/loader/entries/arch.conf** with the text editor of choice and add the following to the file. Note that **intel-ucode.img** is used in this example, replace it with **amd-ucode.img** in amd systems.
```
title Arch Linux
linux vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
```

Write and exit the file and execute this command to add the partition to boot. This should be the root partition, hence, replace **/dev/xxx3** with the root partition of the system.
```
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/xxx3) rw" >> /boot/loader/entries/arch.conf
```

## Setup aur with yay
```
su myusername
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
exit
```
```
pacman -Sy
```

## Setting up networkmanager
This is the perfect time to install the wireless driver. On my laptop the **broadcom-wl-dkms** package works, but please use Google to find the appropriate driver. The dkms version of the driver will always rebuilt for new kernels. If a non-dkms driver was used it would render the driver useless after a kernal upgrade.

Install NetworkManager and enable it on boot.
```
pacman -S networkmanager
systemctl enable NetworkManager
```

## Installing display manager and desktop environment 
In this example the following packages are installed.\
Display manager: sddm\
Desktop environment: kde plasma\
Browsers: firefox, chromium\
Terminal: termite\
File managers: dolphin
```
pacman -S sddm plasma-meta dolphin dolphin-plugins firefox chromium termite
```

Enable sddm on boot
```
systemctl enable sddm
```

## Reboot
```
exit
umount -R /mnt
reboot
```
