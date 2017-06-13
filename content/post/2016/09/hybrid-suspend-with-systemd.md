+++
date = "2016-09-29T11:37:34+01:00"
tags = [ "linux", "systemd" ]
title = "Hybrid suspend with systemd"
+++

[Hybrid suspend] is a suspend mode, where suspend-to-disk and suspend-to-ram are executed together in the same time.
Its a quite usefull mode for notebooks:

- fast wake up because of suspend-to-ram
- no data loss in case of empty battery during the suspend

<!--more-->

Last weekend I updated my laptop to Mint 18 (Ubuntu 16.04 is the base) and noticed the [pm-utils configuration] from Mint 17 (Ubuntu 14.04) doesn't work anymore. The reason is the change towards systemd: systemd is taking care about suspending and hibernating and pm-utils are not used anymore.

In order to get a similar behaviour with systemd and to have hybrid suspend, you should override the suspend target configuration:

```bash
$ cat > /etc/systemd/system/suspend.target << EOF
[Unit]
Description=Hybrid suspend
DefaultDependencies=no
BindsTo=systemd-hybrid-sleep.service
After=systemd-hybrid-sleep.service
Documentation=man:systemd.special(7)
EOF
$ systemctl daemon-reload
```

Thats all :-)

[Hybrid suspend]: http://askubuntu.com/questions/219141/what-is-hybrid-suspend
[pm-utils configuration]: http://askubuntu.com/questions/145443/how-do-i-use-pm-suspend-hybrid-by-default-instead-of-pm-suspend/145676#145676
