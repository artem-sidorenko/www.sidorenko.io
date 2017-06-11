+++
date = "2011-04-27T20:10:00+01:00"
tags = [ "linux", "network", "security", "ipv6" ]
title = "How to activate privacy extensions with IPv6"
+++

If you are using auto-configuration with IPv6, the last part of your IPv6 address will be usually your MAC address. So its possible to create mobility profiles on the simple way. To prevent this you can activate privacy extensions, documented in [RFC3041](http://www.ietf.org/rfc/rfc3041.txt). After you've done it, the last part of your IPv6 address will be random generated

<!--more-->

# Configuration
Place following lines to `/etc/sysctl.conf`

```
#IPv6 privacy extensions
net.ipv6.conf.all.use_tempaddr=2
net.ipv6.conf.default.use_tempaddr=2
net.ipv6.conf.wlan0.use_tempaddr=2
net.ipv6.conf.eth0.use_tempaddr=2
```

Reload sysctl with

```bash
$ sysctl -p /etc/sysctl.conf
```

# Possible issues

As described in the RFC in the part 3.4, your temporary IP are changing by itself during the runtime.
So your active TCP connections will be broken. To avoid this, try to use the permanent IP address for such scenarios.

```bash
# example for ssh client
$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 01:01:01:04:04:05
          ...
          #it's a permanent IP
          inet6 addr: 2001:0001:0001:0:0101:01ff:fe04:0405/64 Scope:Global
          inet6 addr: fe80::0101:01ff:fe04:0405/64 Scope:Link
          #it's a temp IP
          inet6 addr: 2001:0001:0001:0:80b9:1081:2661:80f3/64 Scope:Global
          ...
$ ssh -b 2001:0001:0001:0:0101:01ff:fe04:0405 mysshserver
```

# See also

- [IPv6: Privacy Extensions einschalten (German)](http://www.heise.de/netze/artikel/IPv6-Privacy-Extensions-einschalten-1204783.html)
