+++
date = "2011-01-04T20:46:00+01:00"
tags = [ "linux", "network", "security", "wireshark" ]
title = "How to wireshark with user rights"
+++

It's often nessesary to run wireshark with user rights.

<!--more-->

# Installation

Install wireshark, for Fedora&RH Based distributions:

```bash
yum install wireshark-gnome
```

Create a new user group

```bash
groupadd wireshark
```

Adding the users to the new group: edit /etc/group or use gpasswd

```text
wireshark:x:6668:user1,user2
```

# Permissions of dumpcap
change the permissions and owner of dumpcap

```bash
chown root:wireshark `which dumpcap`
chmod 6550 `which dumpcap`
```

# Change the startup procedure

In RH Based distros consolehelper is used as wrapper to prompt for root password for applications, which need root permissions.

```bash
ls -l /usr/bin/wireshark
```

We don't need it anymore, so we change this symlink.

```bash
unlink /usr/bin/wireshark
ln -s /usr/sbin/wireshark /usr/bin/wireshark
```

From now we start the right wireshark application

# See also

- [http://wiki.wireshark.org/CaptureSetup/CapturePrivileges](http://wiki.wireshark.org/CaptureSetup/CapturePrivileges)
- [http://wiki.wireshark.org/Development/PrivilegeSeparation](http://wiki.wireshark.org/Development/PrivilegeSeparation)
