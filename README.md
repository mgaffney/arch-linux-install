# Arch Linux Install

Install Arch Linux with encrypted file-system and UEFI.
Instructions are for a Dell Precision 5530 (laptop)
using a wired/ethernet connection during
the installation and initial configuration of the system.
WiFi access is not enabled until the __Post Install__ phase.

This is based on:

- https://gist.github.com/mattiaslundberg/8620837
- https://gist.github.com/mjnaderi/28264ce68f87f52f2cabb823a503e673
- https://wiki.archlinux.org/index.php/Installation_guide
- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
- https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption#LVM_on_LUKS

## Create USB installation drive

The Dell laptop comes with Ubuntu pre-installed.
Use it to [download][download] the installation ISO from Arch Linux.
Verify the signature of the downloaded ISO.
Copy to a usb-drive with:

```
dd if=archlinux.img of=/dev/sdb bs=16M && sync
```

[download]: https://archlinux.org/download/

## Prepare disk for install

The disk drive needs to be securely wiped before install.
The main reason is to [prevent disclosure of usage patterns on the
encrypted drive][1].
This also has the added benefit of deleting
and reclaiming the space of
the multiple useless partitions
created by Dell.

The steps for preparing the disk are:

1. Boot from USB
2. Securely wipe the drive
3. Reboot from USB

[1]: https://wiki.archlinux.org/index.php/Disk_encryption#Preparing_the_disk

### Boot from USB

1. Insert the USB and reboot the machine.
1. When the Dell is shown, press `F12` (multiple times) to bring up the
   boot select screen.
1. Select `UEFI BOOT` on the boot select screen.
1. When the Arch menu appears, select `Arch Linux archiso x86_64 UEFI
   CD` (first option) and press `e`.
1. Press `CTRL e` and append `video=1600x900` then press `Enter`.

### Securely wipe the drive

This is a simple, effective and fast method for securely wiping the
existing drive.
More details on this method can be found [here][2]
on the Arch Linux wiki.

```console
root@archiso ~ # cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped
root@archiso ~ # dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=32M
512 GB copied, 396.428 s, 1.3 GB/s
root@archiso ~ # cryptsetup close to_be_wiped
```

This took about ~7 mins for a 512 GB drive.

[2]: https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_specific_methods

### Reboot from USB

Run the following then follow the steps from **Boot from USB**:

```console
root@archiso ~ # reboot now
```

### Update the system clock

```console
root@archiso ~ # timedatectl set-ntp true
```

To check the service status, use `timedatectl status`.

### Partition the disk

We are going to create 3 partitions using [sgdisk][].

[sgdisk]: https://www.rodsbooks.com/gdisk/sgdisk.html

```console
root@archiso ~ # sgdisk \
	-n 1:0:+512M -t 1:ef00 -c 1:"EFI" \
	-n 2:0:+512M -t 2:8300 -c 2:"Boot" \
	-n 3:0:0     -t 3:8e00 -c 3:"Encrypted" \
	-p /dev/nvme0n1
```

### Format the non-encrypted partitions

```console
root@archiso ~ # mkfs.vfat -F32 /dev/nvme0n1p1
root@archiso ~ # mkfs.ext4 /dev/nvme0n1p2
```

### Setup the encryption

```console
root@archiso ~ # cryptsetup luksFormat --type luks2 /dev/nvme0n1p3
root@archiso ~ # cryptsetup open /dev/nvme0n1p3 cryptlvm
```

### Create the logical volumes on the encrypted partition

This creates a root volume and a swap volume in the encrypted partition.

```console
root@archiso ~ # pvcreate /dev/mapper/cryptlvm
root@archiso ~ # vgcreate vg0 /dev/mapper/cryptlvm
root@archiso ~ # lvcreate -L 20G vg0 -n swap
root@archiso ~ # lvcreate -l 100%FREE vg0 -n root
```

Ubuntu recommends the swap size should be equal to the size of RAM plus
the square root of the RAM if hibernation is used.
This follows that recommendation given the laptop will need to
use hibernation.

### Format the LVM partitions

```console
root@archiso ~ # mkfs.ext4 /dev/vg0/root
root@archiso ~ # mkswap /dev/vg0/swap
```

### Mount the new filesystems

```console
root@archiso ~ # mount /dev/vg0/root /mnt
root@archiso ~ # swapon /dev/vg0/swap
root@archiso ~ # mkdir /mnt/boot
root@archiso ~ # mount /dev/nvme0n1p2 /mnt/boot
root@archiso ~ # mkdir /mnt/boot/efi
root@archiso ~ # mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Installation

Install the base packages

```console
root@archiso ~ # pacstrap /mnt base base-devel grub-efi-x86_64 efibootmgr zsh vim
```

### Workaround for grub-mkconfig hanging (part 1)

This is the first part of a workaround.
See [here][b1] and [here][b2] for more details.

```console
root@archiso ~ # mkdir /mnt/hostrun
root@archiso ~ # mount --bind /run /mnt/hostrun
```

[b1]: https://bbs.archlinux.org/viewtopic.php?id=242989
[b2]: https://unix.stackexchange.com/questions/105389/arch-grub-asking-for-run-lvm-lvmetad-socket-on-a-non-lvm-disk/152277#152277

## Configure the system

1. Fstab

	```console
	root@archiso ~ # genfstab -U /mnt >> /mnt/etc/fstab
	```

	(Optional) To make /tmp a ramdisk, add the following line to /mnt/etc/fstab:

	```
	tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0
	```

1. Enter the new system

	```console
	root@archiso ~ # arch-chroot /mnt /bin/bash
	```

1. Workaround for grub-mkconfig hanging (part 2)

	```console
	[root@archiso /]# mkdir /run/lvm
	[root@archiso /]# mount --bind /hostrun/lvm /run/lvm
	```


1. Set the time zone and adjust the clock

	```console
	[root@archiso /]# ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
	[root@archiso /]# hwclock --systohc --utc
	```

1. Configure Localization

	Uncomment `en_US.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`,
	then do the following:

	```console
	[root@archiso /]# vim /etc/locale.gen (uncomment en_US.UTF-8 UTF-8)
	[root@archiso /]# locale-gen
	[root@archiso /]# echo LANG=en_US.UTF-8 > /etc/locale.conf
	[root@archiso /]# export LANG=en_US.UTF-8
	```

1. Configure network

	Create the `/etc/hostname` file:

	```console
	[root@archiso /]# echo myhostname > /etc/hostname
	```

	Add matching entries to `/etc/hosts`:

	```
	127.0.0.1	localhost
	::1		localhost
	127.0.1.1	myhostname.localdomain	myhostname
	```

1. Set root password

	```console
	[root@archiso /]# passwd
	```

1. Create User

	```console
	[root@archiso /]# useradd -m -g users -G wheel -s /bin/zsh mgaffney
	[root@archiso /]# passwd mgaffney
	[root@archiso /]# visudo  #uncomment %wheel ALL=(ALL) ALL
	```

1. Configuring mkinitcpio

	```console
	[root@archiso /]# vim /etc/mkinitcpio.conf
	```

	Add 'ext4' to MODULES.
	Add `keyboard`, `encrypt` and `lvm2` to `HOOKS` before `filesystems`.

	```
	MODULES=(ext4)
	HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)
	```

	Regenerate initrd image:

	```console
	[root@archiso /]# mkinitcpio -p linux
	```

1. Setup grub

	```console
	[root@archiso /]# grub-install
	[root@archiso /]# vim /etc/default/grub
	```

	Edit the following lines to:

	```
	GRUB_CMDLINE_LINUX="cryptdevice=/dev/nvme0n1p3:cryptlvm:allow-discards video=1600x900"
	...
	GRUB_GFXMODE=1600x900x32,1600x900,auto
	```

	then run:

	```console
	[root@archiso /]# grub-mkconfig -o /boot/grub/grub.cfg
	[root@archiso /]# umount /run/lvm
	```

1. Exit new system and unmount all partitions

	```console
	[root@archiso /]# exit
	root@archiso ~ # umount -R /mnt
	root@archiso ~ # swapoff -a
	```

1. Reboot into the new system, don't forget to remove the cd/usb

	```console
	root@archiso ~ # shutdown now
	```

## Post Install

All of the following steps assume you are logged in as root.

### Configure Network

The following steps will configure the laptop to:

- start/stop using an ethernet connection when a cable is plugged
  in/unplugged.
- start/stop using a wifi access point when the laptop enters/leaves the
  range of the access point

__Verify the laptop is still connected to the ethernet cable.__

1. Connect to the network to download and install additional packages

	```console
	[root@hostname ~]# pacman -S dhcpcd
	[root@hostname ~]# systemctl enable dhcpcd.service
	[root@hostname ~]# systemctl start dhcpcd.service
	[root@hostname ~]# pacman -S ifplugd wpa_actiond
	[root@hostname ~]# systemctl stop dhcpcd.service
	[root@hostname ~]# systemctl disable dhcpcd.service
	```

1. Configure ethernet connection

	See [here][netctl] and [here][netcfg] for more details.

	```console
	[root@hostname ~]# cd /etc/netctl
	[root@hostname netctl]# cp examples/ethernet-dhcp .
	[root@hostname netctl]# vim ethernet-dhcp
	```

	Edit the following lines:

	```
	Interface=enp58s0u1
	Priority=2
	```

	```console
	[root@hostname netctl]# systemctl enable netctl-ifplugd@enp58s0u1.service
	[root@hostname netctl]# systemctl start netctl-ifplugd@enp58s0u1.service
	```

[netctl]: https://wiki.archlinux.org/index.php/netctl
[netcfg]: https://wiki.archlinux.org/index.php/Network_configuration

1. Configure Wifi connection(s)

	See [here][netctl-wifi], [here][wifi] and [here][wifisup] for more
	details.

	```console
	[root@hostname ~]# cd /etc/netctl
	[root@hostname netctl]# cp examples/wireless-wpa home-wifi
	[root@hostname netctl]# vim home-wifi
	```

	Edit the following lines:

	```
	Interface=wlp59s0
	ESSID='my-home-essid'
	Key='super-secret-password'
	```

	```console
	[root@hostname netctl]# systemctl enable netctl-auto@wlp59s0.service
	[root@hostname netctl]# systemctl start netctl-auto@wlp59s0.service
	```

[netctl-wifi]: https://wiki.archlinux.org/index.php/netctl#Wireless
[wifi]: https://wiki.archlinux.org/index.php/Wireless_network_configuration
[wifisup]: https://wiki.archlinux.org/index.php/WPA_supplicant

### Update the system

```console
[root@hostname ~]# pacman -Syu
```

### Enable system clock synchronization

```console
[root@hostname ~]# systemctl enable systemd-timesyncd.service
[root@hostname ~]# systemctl start systemd-timesyncd.service
[root@hostname ~]# timedatectl status # to verify
```

## Missing

- VPN
