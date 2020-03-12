# Arch Linux Full-Disk Encryption Installation Guide
# This Document is not finished
- [x] Finished Base Installation
- [X] Translate German to English
- [ ] GNOME and Additionals 
## Warning
This Document is currently work in Progress. I want to provide a full guide from scratch to gnome3.

This guide provides instructions for an Arch Linux installation featuring full-disk encryption via LVM on LUKS and an encrypted boot partition (GRUB) for UEFI systems.

You can Download Arch from [Archlinux.org](https://www.archlinux.org/download/).

## Preface
You will find most of this information pulled from the [Arch Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB)) and other resources linked thereof.

*Note:* The system was installed on an NVMe SSD, substitute ```/dev/nvme0nX``` with ```/dev/sdX``` or your device as needed.

## Pre-installation
### Connect to the internet
Plug in your Ethernet and go, or for wireless consult the all-knowing [Arch Wiki](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).

### Check Network Connection

``` bash
ping google.de
```

### Setting up the keyboard layout
In my case im going to use the German layout. This step important because you could end up with a wrong encryption password.

``` bash
loadkeys de-latin1
```

### Update the system clock
Make sure your BIOS clock is set to UTC. The System time will be calculated of that. 

``` bash
timedatectl set-ntp true
```

### Preparing the disk
#### Create EFI System and Linux LUKS partitions
##### Create a 1MiB BIOS boot partition at start just in case it is ever needed in the future

Number | Start (sector) | End (sector) |    Size    | Code |        Name         |
-------|----------------|--------------|------------|------|---------------------|
   1   |   2048         |   4095       | 1024.0 KiB | EF02 | BIOS boot partition |
   2   |   4096         |   1130495    | 550.0 MiB  | EF00 | EFI System          |
   3   |   1130496      |   976773134  | 465.2 GiB  | 8309 | Linux LUKS          |

```gdisk /dev/nvme0n1```
``` bash
o
n
[Enter]
0
+1M
ef02
n
[Enter]
[Enter]
+550M
ef00
n
[Enter]
[Enter]
[Enter]
8309
w
```

#### Create the LUKS1 encrypted container on the Linux LUKS partition (GRUB does not support LUKS2 as of May 2019)
``` bash
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p3
```

#### Open the container (decrypt it and make available at /dev/mapper/cryptlvm)
``` bash
cryptsetup open /dev/nvme0n1p3 cryptlvm
```

### Preparing the logical volumes
#### Create physical volume on top of the opened LUKS container
``` bash
pvcreate /dev/mapper/cryptlvm
```

#### Create the volume group and add physical volume to it
``` bash
vgcreate vg /dev/mapper/cryptlvm
```

#### Create logical volumes on the volume group for swap, root, and home
``` bash
lvcreate -L 8G vg -n swap
lvcreate -L 32G vg -n root
lvcreate -l 100%FREE vg -n home
```

Example for the correct SWAP Size [itsFOSS](https://itsfoss.com/swap-size/): 

RAM Size	 | Swap Size (Without Hibernation)	 | Swap size (With Hibernation) |
----------|-----------------------------------|------------------------------|
   4GB    |   2GB                             |   6GB                        |
   8GB    |   3GB                             |   11GB                       |
   16GB   |   4GB                             |   20GB                       |

#### Format filesystems on each logical volume
``` bash
mkfs.ext4 /dev/vg/root
mkfs.ext4 /dev/vg/home
mkswap /dev/vg/swap
```

#### Mount filesystems
``` bash
mount /dev/vg/root /mnt
mkdir /mnt/home
mount /dev/vg/home /mnt/home
swapon /dev/vg/swap
```

### Preparing the EFI partition
#### Create FAT32 filesystem on the EFI system partition
``` bash
mkfs.fat -F32 /dev/nvme0n1p2
```

#### Create mountpoint for EFI system partition at /efi for compatibility with grub-install and mount it
``` bash
mkdir /mnt/efi
mount /dev/nvme0n1p2 /mnt/efi
```

## Installation
### Install necessary packages
``` bash
pacstrap /mnt base base-devel linux linux-firmware mkinitcpio lvm2 vim dhcpcd wpa_supplicant netctl
```

## Configure the system
### Generate an fstab file
``` bash
genfstab -U /mnt >> /mnt/etc/fstab
```

#### (optional) Change `relatime` option to `noatime`
```/mnt/etc/fstab```
``` bash
# /dev/mapper/vg-root
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 rw,noatime 0 1

# /dev/mapper/vg-home
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 rw,noatime 0 1
```

Reduces writes to disk when reading from a file, but may cause issues with programs that rely on file access time

### Enter new system chroot
``` bash
arch-chroot /mnt
```

#### At this point you should have the following partitions and logical volumes:
```lsblk```

NAME           | MAJ:MIN | RM  |  SIZE  | RO  | TYPE  | MOUNTPOINT |
---------------|---------|-----|--------|-----|-------|------------|
nvme0n1        |  259:0  |  0  | 465.8G |  0  | disk  |            |
├─nvme0n1p1    |  259:4  |  0  |     1M |  0  | part  |            |
├─nvme0n1p2    |  259:5  |  0  |   550M |  0  | part  | /efi       |
├─nvme0n1p3    |  259:6  |  0  | 465.2G |  0  | part  |            |
..└─cryptlvm   |  254:0  |  0  | 465.2G |  0  | crypt |            |
....├─vg-swap  |  254:1  |  0  |     8G |  0  | lvm   | [SWAP]     |
....├─vg-root  |  254:2  |  0  |    32G |  0  | lvm   | /          |
....└─vg-home  |  254:3  |  0  | 425.2G |  0  | lvm   | /home      |

### Time zone
#### Set the time zone
``` bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

#### Run `hwclock` to generate ```/etc/adjtime```
Assumes hardware clock is set to UTC
``` bash
hwclock --systohc
```

### Localization
#### Uncomment  in ```/etc/locale.gen``` and generate locale
``` bash
de_DE.UTF-8 UTF-8
de_DE ISO-8859-1
de_DE@euro ISO-8859-15
```

#### locales generate

``` bash
local-gen
```

#### Create ```locale.conf``` and set the ```LANG``` variable
```/etc/locale.conf```
``` bash
LANG="de_DE.UTF-8"
LC_COLLATE="C"
LC_TIME="de_DE.UTF-8"
```

### Network configuration and systemd

``` bash
echo myhostname >> /etc/hostname
echo KEYMAP=de-latin1 >> /etc/vconsole.conf
echo FONT=lat9w-16 >> /etc/vconsole.conf
echo FONT_MAP=8859-1_to_uni >> /etc/vconsole.conf
systemctl enable dhcpcd.service
```


#### Add matching entries to hosts
```/etc/hosts```
``` bash
127.0.0.1 localhost
::1 localhost
127.0.1.1 myhostname.localdomain myhostname
```

### Initramfs
#### Add the ```keyboard```, ```encrypt```, and ```lvm2``` hooks to ```/etc/mkinitcpio.conf```
*Note:* ordering matters.
``` bash
HOOKS=(base udev autodetect keyboard modconf block keymap encrypt lvm2 filesystems fsck)
```

#### Recreate the initramfs image
``` bash
mkinitcpio -p linux
```

### Root password
#### Set the root password
``` bash
passwd
```

### Boot loader
#### Install GRUB
``` bash
pacman -S grub
```

#### Configure GRUB to allow booting from /boot on a LUKS1 encrypted partition
```/etc/default/grub```
``` bash
GRUB_ENABLE_CRYPTODISK=y
```

#### Set kernel parameter to unlock the LVM physical volume at boot using ```encrypt``` hook
##### UUID is the partition containing the LUKS container

You need to note or copy your UUID for the GRUP configuration. 

```blkid```
``` bash
/dev/nvme0n1p3: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS" PARTLABEL="Linux LUKS" PARTUUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

```/etc/default/grub```
``` bash
GRUB_CMDLINE_LINUX="... cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:cryptlvm root=/dev/vg/root ..."
```
**Heads Up** The ... at the begining and end of the line are just placeholders. In most cases GRUB_CMDLINE_LINUX="" is empty before the configuration.

#### Install GRUB to the mounted ESP for UEFI booting
``` bash
pacman -S efibootmgr
grub-install --target=x86_64-efi --efi-directory=/efi
```

#### Enable microcode updates
##### grub-mkconfig will automatically detect microcode updates and configure appropriately
``` bash
pacman -S intel-ucode
```

Use intel-ucode for Intel CPUs and amd-ucode for AMD CPUs.

#### Generate GRUB's configuration file
``` bash
grub-mkconfig -o /boot/grub/grub.cfg
```

### (recommended) Embed a keyfile in initramfs

This is done to avoid having to enter the encryption passphrase twice (once for GRUB, once for initramfs.)

#### Create a keyfile and add it as LUKS key
``` bash
mkdir /root/secrets && chmod 700 /root/secrets
head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
cryptsetup -v luksAddKey -i 1 /dev/nvme0n1p3 /root/secrets/crypto_keyfile.bin
```

#### Add the keyfile to the initramfs image
```/etc/mkinitcpio.conf```
``` bash
FILES=(/root/secrets/crypto_keyfile.bin)
```

#### Recreate the initramfs image
``` bash
mkinitcpio -p linux
```

#### Set kernel parameters to unlock the LUKS partition with the keyfile using ```encrypt``` hook
```/etc/default/grub```
``` bash
GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/root/secrets/crypto_keyfile.bin"
```

#### Regenerate GRUB's configuration file
``` bash
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Restrict ```/boot``` permissions
``` bash
chmod 700 /boot
```

The installation is now complete. Exit the chroot and reboot.
``` bash
exit
reboot
```

## Post-installation
Your system should now be fully installed, bootable, and fully encrypted.

If you embedded the keyfile in the initramfs image, it should only require your encryption passphrase once to unlock to the system.

### User and Group Selection

Daily work should not be done with the root account, as it is intended for administrative tasks and working with it can be dangerous. Therefore a normal user is now added. Note that user names may only contain lower case letters and special characters:

In this example the user is called **duda**


``` bash
useradd -m -g users -s /bin/bash duda
passwd duda
```

Another important tool will be installed to execute a command with root privileges:

``` bash
pacman -S sudo
```

In order to give the user root rights, a configuration must be changed:

``` bash
nano /etc/sudoers
```

Find the following line (located below "## Uncomment to allow members of group wheel to execute any command"):
``` bash
 # %wheel ALL=(ALL) ALL
```
and remove the commentary character and the space.
``` bash
%wheel ALL=(ALL) ALL
```

Add the user to the wheel group
``` bash
gpasswd -a duda wheel
```
In order to give the user rights for audio etc., he can be added to the groups ```audio```, ```video```, ```games```, ```power```.

``` bash
gpasswd -a duda audio
gpasswd -a duda video
gpasswd -a duda games
gpasswd -a duda power
```

### Other useful services
#### SSD TRIM
If the system is running on an SSD that supports TRIM, **fstrim.timer** should be enabled:

``` bash
systemctl enable --now fstrim.timer
```
#### GUI Preperation
Now before we turn to the graphical interface and/or multimedia, is a good time to install and activate a few additional services.

``` bash
pacman -S acpid dbus avahi cups
```

These services must be started explicitly. To do this automatically at boot time, systemd must be instructed to do so. This is done by:

``` bash
systemctl enable acpid
systemctl enable avahi-daemon
systemctl enable org.cups.cupsd.service
```

It is also useful to automatically load a network service for Internet access:

Services like NetworkManager can do this. More information can be found at https://wiki.archlinux.de/title/Daemons and at https://wiki.archlinux.de/title/Daemons/Liste.

### Cronjobs

Some packages create so-called cron jobs. These are commands that are executed automatically at certain times. 

``` bash
pacman -S cronie
systemctl enable --now cronie
```

### Automatic time adjustment

If you want to have the time corrected automatically, you can do this with **systemd-timesyncd**.
``` bash
systemctl enable --now systemd-timesyncd.service
```

The time will be correct after a few seconds.
To see if the time is really correct, you can use the following command:

``` bash
date
```
Then you can correct the hardware clock or also RTC or CMOS clock on the motherboard.

``` bash
hwclock -w
```

# Installation X, GNOME (GUI)

TBD
