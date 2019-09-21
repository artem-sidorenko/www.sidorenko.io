---
title: "Full encrypted Proxmox installation"
date: 2019-09-21T21:56:04+02:00
tags: [ "debian", "security" , "proxmox" ]
---

[Proxmox](https://www.proxmox.com/en/proxmox-ve) is a hypervisor software running on Debian Linux.

This post describes the full encrypted installtion of proxmox hypervisor with unlocking of encryption via ssh.

<!--more-->

## Install debian

- [Download debian 10 installation iso](https://www.debian.org/distrib/netinst) and boot the installer
- Run the installation and choose the `encrypted LVM` partitioning type
- Your final partitioning scheme should look like below
![Partitioning scheme](partitioning_scheme.png)
- For the first boot process you will have to give your password via physical console in order to boot the system
- Login via ssh to the system

## Enabling password unlock via ssh and network

Dropbear SSH implementation is used within initramfs to provide the possibility to unlock the encryption of system during the boot process.

Let's install dropbear and do some configuration.

```bash
# install dropber and add the ssh key which should be able to login to initramfs
$ apt install dropbear
...
# add your authorized_keys to the dropbear-initramfs module
# command before the key allows your to directly type the password without to invoke additional commands
$ cat <<EOF > /etc/dropbear-initramfs/authorized_keys
command="cryptroot-unlock" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSZhDcyw9YZagLrSKlAAa1s/YNO9X2aTUVI8A7rgWLehDASckE2IZzd3G/S7j6RXz8jZRelbO3NuGy+LBXc2aJL04+nw5yq7vwbFjkaL4bwbyZmJSasIRbZc8GduHgAT4/NCr2/+4Pvcyt14JkVquIBTlpqfMBtrCWjL1dLrfcvtpJYlxq21v5BWdTZBV0I6Bq8NtB67ly7LgcygtJe38zNuySVkaIyYu0cJ1snnKZjT6V8IdecT4kjeypAzOQZ52UVHLcYvar7FOgD5kIf9R/TStM9zgDa0OFy/XhB8Tx4KTt9wKbgp+1rjEd0yMvJJpq3Vy5QHCC1fWnAoAJ3B0/
EOF
```

Ssh host keys within initramfs and on the system itself should be the same, otherwise you will be regularly annoyed by ssh client warnings.

In the same time, dropbear has it's own ssh key format which isn't compatible to the OpenSSH of the system. Usually `dropbearconvert` can be used for key conversion from OpenSSH in such cases, however `dropbearconvert` doesn't support the modern [RFC4716](http://www.faqs.org/rfcs/rfc4716.html) format used by OpenSSH and expects the keys to be in the `PEM` format. `ssh-keygen` supports both formats, `RFC4716` and `PEM`, but there is no way to convert the private key between different formats with `ssh-keygen`. So we are going to use `puttygen` for `RFC4716 -> PEM` conversion and `dropbearconvert` for `PEM -> DROPBEAR` conversion.

```bash
# install puttygen
$ apt install putty-tools
...
# convert the keys from RFC4716 to PEM
$ puttygen /etc/ssh/ssh_host_rsa_key -O private-openssh -o ssh_host_rsa_key.pem
$ puttygen /etc/ssh/ssh_host_ecdsa_key -O private-openssh -o ssh_host_ecdsa_key.pem
# convert the keys from PEM to Dropbear format
$ /usr/lib/dropbear/dropbearconvert openssh dropbear ssh_host_rsa_key.pem /etc/dropbear-initramfs/dropbear_rsa_host_key
$ /usr/lib/dropbear/dropbearconvert openssh dropbear ssh_host_ecdsa_key.pem /etc/dropbear-initramfs/dropbear_ecdsa_host_key
# cleanup
$ rm ssh_host_*
```

And some final steps for encryption:
```bash
# rebuild the initramfs to include the keys
$ update-initramfs -u -k all
# and try it out
$ reboot
```

## Install proxmox software

[Official wiki](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Buster) describes the procedure pretty good, here is the summary of commands:
```bash
# add a proper entry to /etc/hosts
$ echo "192.168.67.68 debian" >> /etc/hosts
# add repositories
$ echo "deb http://download.proxmox.com/debian/pve buster pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
$ wget http://download.proxmox.com/debian/proxmox-ve-release-6.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg
$ chmod +r /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg
# update the repos and upgrade the system
$ apt update && apt full-upgrade
# install proxmox
$ apt install proxmox-ve postfix open-iscsi
$ apt remove os-prober
# reboot to the new proxmox kernel
$ reboot
# cleanup
$ apt remove linux-image-amd64 'linux-image-4.19*'
$ update-grub

```

That's it, now go to the `https://youripaddress:8006` to see the proxmox interface.

## See too

- [RFC4716](http://www.faqs.org/rfcs/rfc4716.html)
- [How-to : Convert OpenSSH private keys to RSA PEM](https://federicofr.wordpress.com/2019/01/02/how-to-convert-openssh-private-keys-to-rsa-pem/)
- [Install Proxmox VE on Debian Buster](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Buster)
