+++
date = "2010-10-21T22:21:21+01:00"
tags = [ "centos", "linux", "grub" ]
title = "Using native SATA on systems with IDE emulation"
+++

Some systems have this IDE emulation enabled for compatibility purposes.
IDE module and devices are normally initiated by linux kernel before SATA modules and devices.
So, our SATA device is blocked by IDE kernel module and can't be used as native SATA device. Is also not possible to use this emulated IDE device with DMA, so we have a real performance problem with hard disk I/O on this system.

<!--more-->

The solution for this is realy simple, disable scans of IDE bus within grub kernel configuration:

/boot/grub/grub.conf before

```text
title CentOS (2.6.18-164.el5)
        root (hd0,0)
        kernel /vmlinuz-2.6.18-164.el5 ro root=LABEL=/ rhgb quiet
        initrd /initrd-2.6.18-164.el5.img
```

please configure it like this

```text
title CentOS (2.6.18-164.el5)
        root (hd0,0)
        kernel /vmlinuz-2.6.18-164.el5 ro root=LABEL=/ rhgb quiet hda=noprobe hdb=noprobe hdc=noprobe hdd=noprobe
        initrd /initrd-2.6.18-164.el5.img
```

check the fstab and reboot the system
