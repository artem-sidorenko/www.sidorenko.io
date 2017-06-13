+++
date = "2012-04-05T11:50:00+01:00"
tags = [ "vsphere", "esx", "esxi" ]
title = "How to create VMFS partition on the local disk without label"
+++

If you are going to create the VMFS partition on the some mew local hdd or hardware raid volume, you can get in trouble with a message "Failed to update the disk partition information".
The reason for it is a broken/missing dos label on the device.

<!--more-->

# Create a new label

- Type Alt+F1 on the physical ESXi console
- Type "unsupported" and after a password prompt your root password
- Identify your device with

```bash
esxcfg-scsidevs -l | less
```

- note the Devfs path of the output above, and start fdisk

```bash
fdisk <<here the devfs path, something like /vmfs/devices/disks/...>>
```

- type "o" "Enter" "w" "Enter"
- type "exit"
- Go ahead and try again to create VMFS volume, it should work now

# See too

- [Creating a local VMFS volume generates the error: Error during the configuration of the host: Failed to update the disk partition information](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1001489)
- [Tech Support Mode for Emergency Support](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1003677)
- [Identifying disks when working with VMware ESX](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1014953)
