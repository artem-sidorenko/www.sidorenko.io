+++
date = "2016-12-10T23:16:28+01:00"
tags = [ "ipv6", "mikrotik", "network", "routeros", "fritzbox" ]
title = "IPv6 with prefix delegation, RouterOS and FritzBox"
+++

I have a non-flat network with subnetworks at home and I wanted to enable IPv6 in dual stack mode for the desktop systems.
This blog post describes this setup and configuration for:

- [MikroTik CRS125-24G-1S-IN] layer 3 switch as switch/router for internal networks (RouterOS 6.36.4)
- [AVM FritzBox 7390] as internet router (FRITZ!OS 06.51)
- [DT] as ISP with native IPv6 in dual stack mode and dynamic IPv6 prefixes

<!--more-->

Following picture gives an overview on the network setup with IPv4 and subnetworks:

![IPv4 overview](intro-ipv4-overview.png)

As we want to configure IPv6 in dual stack mode, we are not going to change the IPv4 configuration. We will only add the IPv6 configuration. It will be done in two steps:

- Configuration of ULA for internal communication
- Configuration of GUA for internet communication

# Unique local address

For the internal IPv6 communication we will configure [unique local addresses (ULA)]. ULA is comparable with private addresses of IPv4, but these addresses are intended to be unique to avoid any collisions within ULA address space. It makes sense to register your ULA prefix on the [Sixxs ULA database].
In this blog post we use `2001:db8:fc00::/48` as ULA prefix.

![IPv6 ULA prefixes](ipv6-ula.png)

The configuration on the RouterOS is simple: add two IPv6 addresses to the vlan interfaces:

```bash
[admin@routeros] > /ipv6 address
[admin@routeros] /ipv6 address> add interface=vlan10 address=2001:db8:fc00:10::/64 advertise=yes
[admin@routeros] /ipv6 address> add interface=vlan20 address=2001:db8:fc00:20::/64 advertise=yes
```

# Global unicast addresses

In the dual stack mode, besides a public IPv4 address, you get a /56 IPv6 global unicast address (GUA) prefix assigned by DT. However, this /56 prefix is dynamic: after each reconnection you get a new prefix.
Because of this you can not simply make a static configuration. But there is a solution - [DHCPv6 prefix delegation]. Prefix delegation works like this:

- internal router asks the internet router to assign him part of the /56 GUA prefix via DHCPv6
- the requested prefix can be splitted to the subnetworks and then assigned to the interfaces and announced via router advertisements to the nodes

![IPv6 GUA prefixes](ipv6-gua.png)

Lets assume following things:

- following /56 prefix was assigned by DT: `2001:db8:aaaa:aa00::/56`
- following GUA prefix in the external network was configured: `2001:db8:aaaa:aa00::/64`

Steps for prefix delegation:

- Mikrotik CRS asks FritzBox for /58 prefix delegation and gets following prefix delegated: `2001:db8:aaaa:aa40::/58`
- Mikrotik CRS splits this prefix to /64 subnets, configures the IP addresses on the interfaces and starts the router advertisements of this prefix

Following configuration steps are required on the FritzBox:

- Internet -> Zugangsdaten -> IPv6 -> Unterstützung für IPv6 aktiv -> Immer eine native IPv6-Anbindung nutzen
- Heimnetz -> Heimnetzübersicht -> Netzwerkeinstellungen -> IPv6-Adressen -> DHCPv6-Server in der FRITZ!Box für das Heimnetz aktivieren -> DNS-Server und IPv6-Präfix (IA_PD) zuweisen

Following configuration on the Mikrotik CRS implements the above described steps:

```bash
[admin@routeros] > /ipv6 dhcp-client
[admin@routeros] /ipv6 dhcp-client> add add-default-route=yes interface=vlan50 pool-name=gua-internet prefix-hint=::/58 request=prefix use-peer-dns=yes
[admin@routeros] > /ipv6 address
[admin@routeros] /ipv6 address> add interface=vlan10 from-pool=gua-internet advertise=yes
[admin@routeros] /ipv6 address> add interface=vlan20 from-pool=gua-internet advertise=yes
```

# Side effect of dynamic prefixes

What happens if FritzBox gets a new GUA prefix (reboot of router, disconnect etc)? Mikrotik CRS will notice the new prefix by the expiration of valid lifetime for the old prefix delegation. In my case FritzBox was delegating the prefixes with 2 hours lifetime. So, in worst case Mikrotik CRS won't notice the changed GUA prefix for 2 hours. All nodes behind Mikrotik CRS will keep the old GUA prefix and would not be able to reach the Internet.

I asked AVM about the feature to get the prefix delegation lifetime configurable, hopefully it will get available in one of the next releases. In the meantime you can use following workaround on the Mikrotik CRS:

```bash
[admin@routeros] > /system scheduler
[admin@routeros] /system scheduler> add interval=30s name=refresh-ipv6 on-event="/ipv6 dhcp-client renew [find interface=vlan50]" policy=read,write start-date=nov/28/2016 start-time=18:26:35
```

This script runs all 30 seconds and invokes the DHCPv6 renew. In case of GUA prefix change, the new GUA prefix will get distributed by Mikrotik CRS by the next 30 seconds.

# See too

- [Setting up a IPv6 subnet in the FRITZ!Box]
- [Wikipedia: Prefix Delegation][DHCPv6 prefix delegation]
- [RFC: IPv6 Prefix Options for DHCPv6]

[MikroTik CRS125-24G-1S-IN]: https://routerboard.com/CRS125-24G-1S-IN
[AVM FritzBox 7390]: https://avm.de/service/fritzbox/fritzbox-7390/uebersicht/
[DT]: https://www.telekom.de/
[Sixxs ULA database]: https://www.sixxs.net/tools/grh/ula/
[unique local addresses (ULA)]: https://tools.ietf.org/search/rfc4193
[DHCPv6 prefix delegation]: https://en.wikipedia.org/wiki/Prefix_delegation
[RFC: IPv6 Prefix Options for DHCPv6]: https://tools.ietf.org/html/rfc3633
[Setting up a IPv6 subnet in the FRITZ!Box]: https://en.avm.de/service/fritzbox/fritzbox-7490/knowledge-base/publication/show/1239_Setting-up-a-IPv6-subnet-in-the-FRITZ-Box/
