+++
date = "2015-02-08T14:43:14+01:00"
tags = [ "centos", "security", "linux" ]
title = "Move existing centos installation to encrypted raid and lvm"
+++

If you have already some CentOS/RHEL installation and you want to move it to a RAID/LVM with full disk encryption, then this post is for you. We will move a minimal simple CentOS 7 installation on a single disk with LVM to the full encrypted RAID1&LVM setup.

<!-- more -->

Following steps will be done here:

- RAID creation on the second disk
- Setup of luks
- Setup of LVM
- Data movement
- Reboot from second disk
- Adding the first disk to the RAID

```bash
#install mdadm
$ yum -y install mdadm
#label the second disk
$ fdisk /dev/sdb
#create the raid devices without the first hdd
$ mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdb1 missing
$ mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sdb2 missing
#setup the encryption layer
$ yum -y install cryptsetup
$ cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/md2
$ cryptsetup luksOpen /dev/md2 root
#create LVM
$ pvcreate /dev/mapper/root
$ vgcreate vg /dev/mapper/root
#create the logical volumes (feel free to update this to match your layout)
$ lvcreate -nswap -L8G vg
$ lvcreate -nroot -l100%FREE vg
#create file systems
$ mkswap -f /dev/vg/swap
$ mkfs.ext4 -L/ /dev/vg/root
$ mkfs.ext4 -L/boot /dev/md1
#check the fs directly (otherwise it waill take some time during the first boot)
$ fsck.ext4 /dev/vg/root
$ fsck.ext4 /dev/md1
#mount and copy the data
$ yum -y install rsync
$ mount /dev/vg/root /mnt
$ mkdir /mnt/boot
$ mount /dev/md1 /mnt/boot
$ rsync -aAX /{bin,boot,etc,home,lib,lib64,media,opt,root,sbin,srv,tmp,usr,var} /mnt/
$ mkdir /mnt/{dev,mnt,proc,sys,run}
#chroot in the new environment
$ mount -o bind /dev /mnt/dev
$ mount -o bind /proc /mnt/proc
$ mount -o bind /sys /mnt/sys
$ chroot /mnt bash
$ source /etc/profile
#create crypttab and a list of md raid arrays
$ echo "luks-$(blkid -o value /dev/md2 | head -n1) UUID=$(blkid -o value /dev/md2 | head -n1) none" > /etc/crypttab
$ mdadm --examine --scan > /etc/mdadm.conf
#update the fstab to something like this
$ echo -e "/dev/vg/root\t/\text4\tdefaults\t0\t1">/etc/fstab
$ echo -e "/dev/md1\t/boot\text4\tdefaults\t0\t2">/etc/fstab
$ echo -e "/dev/vg/swap\tswap\tswap\tdefaults\t0\t0">>/etc/fstab
#change the grub settings
$ sed -i "/GRUB_CMDLINE_LINUX/d" /etc/default/grub
$ echo "GRUB_CMDLINE_LINUX=\"rd.lvm.lv=vg/swap crashkernel=auto rd.lvm.lv=vg/root rd.md.uuid=$(grep /dev/md/1 /etc/mdadm.conf | cut -d" " -f5 | cut -d= -f2) rd.md.uuid=$(grep /dev/md/2 /etc/mdadm.conf | cut -d" " -f5 | cut -d= -f2) rd.luks.uuid=luks-$(blkid -o value /dev/md2 | head -n1) vconsole.font=latarcyrheb-sun16 vconsole.keymap=de rhgb quiet\"" >> /etc/default/grub
#ignore lvm related warnings
$ grub2-mkconfig > /etc/grub2.cfg
$ grub2-install /dev/sdb
#regenerate the initramfs
$ dracut --force
#reboot from a second disk
$ exit
$ reboot
#choose the second disk during the boot
#enter the encryption password during the boot

#add first HDD to the raid
$ sfdisk -d /dev/sdb | sfdisk /dev/sda
#in case of 'Device or resource busy' try to remove the old volume group with vgremove [volgroup name]
$ mdadm --manage /dev/md1 --add /dev/sda1
$ mdadm --manage /dev/md2 --add /dev/sda2
#install grub on the first HDD
$ grub2-install /dev/sda
#You can switch the boot order on the next reboot to the first HDD again
```
