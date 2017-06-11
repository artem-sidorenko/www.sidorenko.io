+++
date = "2010-10-22T18:25:00+01:00"
tags = [ "linux", "network", "kickstart", "centos", "redhat", "syslinux", "pxelinux" ]
title = "How to use the correct interface for kickstart installations"
+++

When your system has more than one network interface anaconda asks you which one you'd like to use for the kickstart process. This decision can be made at boot time by adding the ksdevice paramter and setting it accordingly. To run kickstart via eth0 simply add ksdevice=eth0 to the kernel command line.

<!--more-->

The second method is using ksdevice=link. In this case anaconda will use the first interface it finds that has a active link.

The third method works if you are doing PXE based installations. Then you add IPAPPEND 2 to the PXE configuration file and use ksdevice=bootif. In this case anaconda will use the interface that did the PXE boot (this does not necessarily needs to be the first one with a active link).

Sample:

```bash
#Install CentOS 5.4 x86
label install_centos54_x86
menu label ^CentOS 5.4 x86
kernel mirror/CentOS/releases/5/Everything/i386/isolinux/vmlinuz
ipappend 2
append initrd=mirror/CentOS/releases/5/Everything/i386/isolinux/initrd.img ramdisk_size=8192 ip=dhcp ks=http://kickstartserver/configs/centos_54_x86/ks.cfg ksdevice=bootif
```

See too

- [http://readlist.com/lists/centos.org/centos/12/63832.html](http://readlist.com/lists/centos.org/centos/12/63832.html)
- [http://linux.dell.com/files/whitepapers/nic-enum-whitepaper-v3.pdf](http://linux.dell.com/files/whitepapers/nic-enum-whitepaper-v3.pdf)
- [http://wiki.centos.org/TipsAndTricks/KickStart](http://wiki.centos.org/TipsAndTricks/KickStart)
