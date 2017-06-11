+++
date = "2010-10-21T22:13:24+01:00"
tags = [ "linux" ]
title = "How to edit initrd image"
+++

It usefull sometimes to edit the initrd/initramfs image of the system. This howto describes the simple and usual way to do it.

<!-- more -->

**Hint:** Mainstream distributions have own mechanisms to do this things. Take a look at mkinitramfs/mkinitrd. In such case your customized image can be overwritten by this tools, so try to do in on the right way;)

 - Copy to the temporary folder

```bash
cd /tmp
mkdir initrd
cd initrd
cp /boot/initrd* .
```

 - Detect the type of image

```bash
gunzip -c -S "" initrd* > decompressed-initrd
file decompressed-initrd
```

If you get something like "ASCII cpio archive (SVR4 with no CRC)" process with working with CPIO archive,
if you get something like "Linux rev 1.0 ext2 filesystem data" process with working with ext2 image

 - working with CPIO archive
   - unpacking

```bash
rm decompressed-initrd
cat initrd* | gzip -d | cpio -i
rm initrd*
```

or

```bash
rm initrd*
cat decompressed-initrd | cpio -i
rm decompressed-initrd
```

   - packing

```bash
find | cpio -H newc -o | gzip > ../newinitrd
```

 - working with ext2 image
   - mounting

```bash
rm initrd*
mkdir mnt
mount -o loop decompressed-initrd mnt
```

   - unmounting&packing

```bash
umount mnt
rm -rf mnt
cat decompressed-initrd | gzip > ../newinitrd
```
