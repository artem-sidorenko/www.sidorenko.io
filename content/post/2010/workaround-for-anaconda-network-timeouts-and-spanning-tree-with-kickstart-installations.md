+++
date = "2010-11-08T20:01:00+01:00"
tags = [ "cisco", "pxelinux", "network", "linux", "kickstart", "centos", "redhat" ]
title = "Workaround for Anaconda network timeouts and spanning tree with Kickstart installations"
+++

If you are running spanning tree you may get in trouble with anaconda network initialization.
Spanning tree needs about a 30 seconds to bring the switch port up, so you may run in dhcp timeout.

<!--more-->

To solve with add following anaconda option to the kernel boot parameters:

```text
dhcptimeout=60
```

Sample:

```text
#Install CentOS 5.4 x86
label install_centos54_x86
menu label ^CentOS 5.4 x86
menu default
kernel repo/CentOS/releases/5.4/i386/isolinux/vmlinuz
ipappend 2
append initrd=repo/CentOS/releases/5.4/i386/isolinux/initrd.img ramdisk_size=8192 ip=dhcp ks=http://kickstartserver/configs/centos_54_x86/ks.cfg ksdevice=bootif noshell unattended=yes dhcptimeout=60
```

See also:

- [Anaconda Boot Options] (http://fedoraproject.org/wiki/Anaconda/Options)
- [nicdelay and linksleep May Be of Assistance](http://fedoraproject.org/wiki/Anaconda/NetworkIssues#nicdelay_and_linksleep_May_Be_of_Assistance)
