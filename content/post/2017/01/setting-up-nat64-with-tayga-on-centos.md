+++
date = "2017-01-04T21:33:46+01:00"
tags = [ "ipv6", "centos", "network" ]
title = "Setting up NAT64 with tayga on Centos"
+++

Maybe you are also playing aroung with IPv6 and want to setup IPv6 only network and asking yourself how to reach the IPv4 Internet?
Right, with [DNS64] and [NAT64]. This blog post gives an overview about a such setup on CentOS/RHEL 7 with bind and [tayga].

<!--more-->

# Overview

How does it work? DNS server checks if there is an AAAA entry (IPv6 address).
If there is none, but there is an A entry (IPv4 address), DNS server encodes the IPv4 addresses into the configured /96 IPv6 prefix and returns it as AAAA entry.
This /96 prefix is routed to the NAT64 gateway. This gateway makes a translation between IPv6 and IPv4.

The DNS64 is supported by the last bind versions and [tayga] is one of the NAT64 gateway implementations for Linux.

So, you will need a /96 prefix. You can use the reserved prefix `64:ff9b::/96` for this purpose, but keep in mind: you won't be able to use the translation to the private IPv4 addresses defined by [RFC1918].
If you want to be able to reach this addresses too, you should use a ULA prefix (and you can also register it in the [Sixxs ULA database]).
In this blog post I assume that you use the `2001:db8:64:ff9b::/96` prefix.

# DNS64

Configuration of bind is pretty simple and looks like this:

```text
...

acl "ignore" { // do not apply DNS64 to this addresses
  10/8;
  172.16/12;
};

acl "clients" { // your IPv6 clients
  ::1/128;
  2001:db8:1000:/64; // maybe some network with IPv6 clients
};

options {
...
  listen-on-v6 port 53 { any; };

  dns64 2001:db8:64:ff9b::/96 {
    clients { clients; };
    mapped { !ignore; any; };
    break-dnssec yes; // more on this below
  };
...
};
...

```

In this configuration we do not use DNS64 for ranges in the ACL `ignore` (if you are doing [dual-stack], this might be the ranges you want to keep IPv4 only).
The ACL `clients` defines the source range of DNS clients, where DNS64 should be used (might be useful for multi-view setups).

Now some words on the `break-dnssec` option. Jen Linkova [did a good explanation of the problem at the RIPE72][IPv6-Only and DNSSEC64] (and here a short [blog post][Let's talk about IPv6 DNS64 & DNSSEC]).
If we want to be able to reach the IPv4-only systems with enabled DNSSEC for their DNS entries: we have to break DNSSEC. However, this should be a very small amount of systems on the internet (see the numbers on the slides of Jen Linkova).

# NAT64 with tayga

You can install tayga from EPEL repository:

```bash
$ yum -y install epel-release
...
$ yum -y install tayga
...
```

## Basic network configuration of CentOS system

Then you should configure the IPv4 and IPv6 transport networks on the CentOS box, we use `192.0.2.0/30` and `2001:db8:20::/64` for this purpose:

- `192.0.2.1` is the router IP (e.g. your internet router with IPv4 connectivity)
- `192.0.2.2` is the IP of tayga system
- `2001:db8:20::1/64` is the router IP (e.g. you router with IPv6 clients)
- `2001:db8:20::2/64` is the IP of tayga system

```text
# /etc/sysconfig/network-scripts/ifcfg-ens3
DEVICE=ens3
ONBOOT=yes
BOOTPROTO=none
TYPE=Etheret
IPADDR="192.0.2.2"
NETMASK="255.255.255.252"
GATEWAY="192.0.2.1"
IPV6INIT=yes
IPV6FORWARDING=yes
IPV6ADDR=2001:db8:20::2/64
IPV6_DEFAULTGW=2001:db8:20::1/64
```

If you would like to accept RAs on this interface (e.g. in order to provide IPv6 internet to the CentOS system like described in [this blog post](/blog/2016/12/10/ipv6-with-prefix-delegation-and-mikrotik/)),
you will have to create a file `/sbin/ifup-local` and make it executable:

```bash
#!/bin/sh
# /sbin/ifup-local
if [[ "$1" == "ens3" ]];
then
  sysctl net.ipv6.conf.ens3.accept_ra=2
fi
```

The reason for this are the RHEL network scripts (esp. `/etc/sysconfig/network-scripts/ifup-ipv6:93-129`).
You do not have any way to set `net.ipv6.conf.ens3.accept_ra=2` in order to accept RAs (`IPV6_AUTOCONF=yes` sets it to `1` and its not enough in this case).

## Routing configuration

Tayga will need some IPv4 network for address translation. We will use `192.0.2.128/25` for this. So you will need following routing configuration on your router(s):

- IPv4 `192.0.2.128/25` via `192.0.2.2`
- IPv6 `2001:db8:64:ff9b::/96` via `2001:db8:20::2/64`

Enable IPv4 and IPv6 forwarding on the CentOS system:

```bash
$ cat > /etc/sysctl.d/forward.conf <<EOF
> net.ipv4.ip_forward=1
> net.ipv6.conf.all.forwarding=1
> EOF
$ sysctl --system
...
```

## Network configuration for Tayga

Tayga needs its own tun interface, which is used as NAT64 gateway. Create following interface and routing configurations:

```text
# /etc/sysconfig/network-scripts/ifcfg-tunnat64
TYPE=Tap  # Do not be confused about tap instead of tun here. Tun interface type gets determined by the interface name
BOOTPROTO=none
DEVICE=tunnat64
ONBOOT=yes
IPV6INIT=yes
IPV6FORWARDING=yes
```

```text
# /etc/sysconfig/network-scripts/route-tunnat64
192.0.2.128/25 dev tunnat64
```

```text
# /etc/sysconfig/network-scripts/route6-tunnat64
2001:db8:64:ff9b::/96 dev tunnat64
```

and now enable the tunnel interface:

```bash
$ ifup tunnat64
```

## Tayga configuration

Create the tayga configuration:

```text
# /etc/tayga/default.conf
tun-device tunnat64
ipv4-addr 192.0.2.129
prefix 2001:db8:64:ff9b::/96
dynamic-pool 192.0.2.128/25
data-dir /var/lib/tayga/default
```

and start tayga:

```bash
$ systemctl enable tayga@default
$ systemctl start tayga@default
```

# See too

- [NAT64 Setup Using Tayga]
- [A quick NAT64/DNS64 setup]
- [BIND 9 configuration reference](https://ftp.isc.org/isc/bind/9.8.0-P4/doc/arm/Bv9ARM.ch06.html)
- [IPv6-Only and DNSSEC64]
- [Let's talk about IPv6 DNS64 & DNSSEC]

[NAT64]: https://en.wikipedia.org/wiki/NAT64
[DNS64]: https://en.wikipedia.org/wiki/IPv6_transition_mechanism#DNS64
[NAT64 Setup Using Tayga]: http://packetpushers.net/nat64-setup-using-tayga/
[A quick NAT64/DNS64 setup]: http://ipvsix.me/?p=106
[RFC1918]: https://tools.ietf.org/html/rfc1918
[Sixxs ULA database]: https://www.sixxs.net/tools/grh/ula/
[tayga]: http://www.litech.org/tayga/
[IPv6-Only and DNSSEC64]: https://ripe72.ripe.net/presentations/152-IPv6-Only-DNS64-and-DNSSEC4.pdf
[Let's talk about IPv6 DNS64 & DNSSEC]: https://blog.apnic.net/2016/06/09/lets-talk-ipv6-dns64-dnssec/
[dual-stack]: https://en.wikipedia.org/wiki/IPv6#Dual_IP_stack_implementation
