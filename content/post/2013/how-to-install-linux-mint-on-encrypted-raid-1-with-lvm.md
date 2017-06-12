+++
date = "2013-12-31T18:00:48+01:00"
tags = [ "linux", "mint", "ubuntu" ]
title = "How to install Linux Mint on encrypted raid 1 with LVM"
+++

The installer of Linux Mint doesn't support the installation on encrypted raid 1 with LVM out of the box.

Following steps are required to do this with Linux Mint 16 (without GPT&UEFI)

<!-- more -->

- Boot your system with a Linux Mint LiveCD
- Configure network to get internet
- Open a shell and get root via `sudo -i bash`

```bash
#install mdadm
$ apt-get install mdadm
#label the first hdd, we need at 2 partitions:
# first one for /boot about 500MB and active for boot
# second one for encrypted LVM, all free space
$ fdisk /dev/sda
#copy the label to the second hdd
$ sfdisk -d /dev/sda | sfdisk /dev/sdb
#create the raids
$ mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=0.90 /dev/sda1 /dev/sdb1
$ mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sda2 /dev/sdb2
#create and open the encryption layer
$ cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/md1
$ cryptsetup luksOpen /dev/md1 root
#create LVM
$ pvcreate /dev/mapper/root
$ vgcreate vg /dev/mapper/root
#create the labels
$ lvcreate -nroot -L50G vg
$ lvcreate -nvar-log -L10G vg
$ lvcreate -nswap -L8G vg
$ lvcreate -ntmp -L10G vg
$ lvcreate -nhome -L100G vg
```

- start now the installer via shell to disable the boot loader installation ([it would fail](https://github.com/linuxmint/ubiquity/issues/24), on Mint 18 it would even totally break the installer)

```bash
$ ubiquity -b
```

- select the labels in the partitioning steps
- when installer is done, do the following steps

```bash
#mount the system
$ mount /dev/vg/root /target
$ mount /dev/vg/home /target/home
$ mount /dev/vg/tmp /target/tmp/
$ mount /dev/vg/var-log /target/var/log
$ mount -o bind /proc /target/proc
$ mount -o bind /sys /target/sys
$ mount -o rbind /dev /target/dev
$ mount /dev/md0 /target/boot/
#change into the installed system
$ chroot /target/
#configure dns
$ echo "nameserver 8.8.8.8">>/etc/resolv.conf
#install the packages
$ apt-get install mdadm
#create the crypttab
$ echo "root /dev/md1 none luks">/etc/crypttab
#reinstall dpkg and upstart to avoit some bug
$ apt-get install --reinstall dpkg upstart
#update the initramfs
$ update-initramfs -u -k all
#install grub
$ update-grub
$ grub-install /dev/sda
$ grub-install /dev/sdb
#reboot
$ exit
$ reboot
```

See too:

- [Raid 1 on mint 12](http://forums.linuxmint.com/viewtopic.php?f=46&t=89272#p514496)
- [Re: Mint 14 64bit fresh install on software raid1 loops forever](http://forums.linuxmint.com/viewtopic.php?f=46&t=117699#p669558)
