+++
date = "2014-03-20T12:27:37+01:00"
tags = [ "security", "linux", "security" ]
title = "Default iptables rules"
+++

I use following iptables and ip6tables rules as a default. This rules provide basic security level:

- statefull inspection
- port knocking for management services like ssh
- port scan protection

<!--more-->

# Requirements

- You should have CONFIG_NETFILTER_XT_MATCH_RECENT enabled in the kernel
- You should have CONFIG_NETFILTER_XT_MATCH_CONNTRACK enabled in the kernel

# Overview

## Port scan protection

Port scan protection works with a honeypot on the telnet and samba ports. If some system hit one of this ports(tcp 23,139,445 udp 137,138) it gets completly blocked till absolutely no traffic comes from this source for 10 minutes.

## Port knocking

You will be able to access ssh, only if you send new tcp packets to the ports 8881->7777->9991->9992, exactly in this order. Knocking and connection itself have to be initiated within 1 minute.

# Rules

Following rules can be loaded with iptables-restore or ip6tables-restore

```text
# iptables.rules
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]

#honeypot chain for portscans
-N HONEYPOT
-A HONEYPOT -m recent --update --seconds 600 --name HONEYPOT --reap --rttl -j DROP
-A HONEYPOT -p tcp -m tcp -m multiport --dport 23,139,445 -m recent --name HONEYPOT --set -j LOG --log-prefix "HONEYPOT: " --log-level 6 --log-ip-options
-A HONEYPOT -p udp -m udp -m multiport --dport 137,138 -m recent --name HONEYPOT --set -j LOG --log-prefix "HONEYPOT: " --log-level 6 --log-ip-options
-A HONEYPOT -p tcp -m tcp -m multiport --dports 23,139,445 -m recent --name HONEYPOT --set -j DROP
-A HONEYPOT -p udp -m udp -m multiport --dports 137,138 -m recent --name HONEYPOT --set -j DROP

#port knocking chains, so knocking has to be done 8881->7777->9991->9992
-N PORTKNOCKP1
-A PORTKNOCKP1 -m recent --name PORTKNOCKP1 --set
#-A PORTKNOCKP1 -j LOG --log-prefix "PORTKNOCK PHASE1: "
-N PORTKNOCKP2
-A PORTKNOCKP2 -m recent --name PORTKNOCKP1 --remove
-A PORTKNOCKP2 -m recent --name PORTKNOCKP2 --set
#-A PORTKNOCKP2 -j LOG --log-prefix "PORTKNOCK PHASE2: "
-N PORTKNOCKP3
-A PORTKNOCKP3 -m recent --name PORTKNOCKP2 --remove
-A PORTKNOCKP3 -m recent --name PORTKNOCKP3 --set
#-A PORTKNOCKP3 -j LOG --log-prefix "PORTKNOCK PHASE3: "
-N PORTKNOCKP4
-A PORTKNOCKP4 -m recent --name PORTKNOCKP3 --remove
-A PORTKNOCKP4 -m recent --name PORTKNOCKP --set
#-A PORTKNOCKP4 -j LOG --log-prefix "PORTKNOCK PHASE4: "

-N PORTKNOCK
-A PORTKNOCK -p tcp -m tcp --dport 8881 -m conntrack --ctstate NEW -j PORTKNOCKP1
-A PORTKNOCK -p tcp -m tcp --dport 7777 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP1 -j PORTKNOCKP2
-A PORTKNOCK -p tcp -m tcp --dport 9991 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP2 -j PORTKNOCKP3
-A PORTKNOCK -p tcp -m tcp --dport 9992 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP3 -j PORTKNOCKP4

#allow loopback
-A INPUT -i lo -j ACCEPT
#pass it to the honeypot
-A INPUT ! -s 127.0.0.1 -j HONEYPOT
#allow established communication
-A INPUT -p tcp -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p udp -m conntrack --ctstate ESTABLISHED -j ACCEPT
-A INPUT -p icmp -m conntrack --ctstate ESTABLISHED -j ACCEPT

#allow ping
-A INPUT -p icmp -m conntrack --ctstate NEW -m icmp --icmp-type 8 -j ACCEPT
#allow public services
-A INPUT -p tcp -m tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT
#check portknocking and allow protected services
-A INPUT ! -s 127.0.0.1 -j PORTKNOCK
-A INPUT -p tcp -m tcp -m multiport --dports 22 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --name PORTKNOCK --reap -j ACCEPT

#allow outgoing communication
-A OUTPUT -o lo -j ACCEPT
-A OUTPUT -p tcp -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -p udp -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -p icmp -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT

COMMIT
```

```text
# ip6tables.rules
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]

#IPv6 specific stuff, neighbor solicitation, advertisement
-A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 135 -j ACCEPT
-A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 136 -j ACCEPT

#honeypot chain for portscans
-N HONEYPOT
-A HONEYPOT -m recent --update --seconds 600 --name HONEYPOT --reap --rttl -j DROP
-A HONEYPOT -p tcp -m tcp -m multiport --dports 23,139,445 -m recent --name HONEYPOT --set -j LOG --log-prefix "HONEYPOT: " --log-level 6 --log-ip-options
-A HONEYPOT -p udp -m udp -m multiport --dports 137,138 -m recent --name HONEYPOT --set -j LOG --log-prefix "HONEYPOT: " --log-level 6 --log-ip-options
-A HONEYPOT -p tcp -m tcp -m multiport --dports 23,139,445 -m recent --name HONEYPOT --set -j DROP
-A HONEYPOT -p udp -m udp -m multiport --dports 137,138 -m recent --name HONEYPOT --set -j DROP

#port knocking chains, so knocking has to be done 8881->7777->9991->9992
-N PORTKNOCKP1
-A PORTKNOCKP1 -m recent --name PORTKNOCKP1 --set
#-A PORTKNOCKP1 -j LOG --log-prefix "PORTKNOCK PHASE1: "
-N PORTKNOCKP2
-A PORTKNOCKP2 -m recent --name PORTKNOCKP1 --remove
-A PORTKNOCKP2 -m recent --name PORTKNOCKP2 --set
#-A PORTKNOCKP2 -j LOG --log-prefix "PORTKNOCK PHASE2: "
-N PORTKNOCKP3
-A PORTKNOCKP3 -m recent --name PORTKNOCKP2 --remove
-A PORTKNOCKP3 -m recent --name PORTKNOCKP3 --set
#-A PORTKNOCKP3 -j LOG --log-prefix "PORTKNOCK PHASE3: "
-N PORTKNOCKP4
-A PORTKNOCKP4 -m recent --name PORTKNOCKP3 --remove
-A PORTKNOCKP4 -m recent --name PORTKNOCK --set
#-A PORTKNOCKP4 -j LOG --log-prefix "PORTKNOCK PHASE4: "

-N PORTKNOCK
-A PORTKNOCK -p tcp -m tcp --dport 8881 -m conntrack --ctstate NEW -j PORTKNOCKP1
-A PORTKNOCK -p tcp -m tcp --dport 7777 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP1 -j PORTKNOCKP2
-A PORTKNOCK -p tcp -m tcp --dport 9991 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP2 -j PORTKNOCKP3
-A PORTKNOCK -p tcp -m tcp --dport 9992 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --reap --name PORTKNOCKP3 -j PORTKNOCKP4

#allow loopback
-A INPUT -i lo -j ACCEPT
#pass traffic to the honeypot
-A INPUT ! -s ::1 -j HONEYPOT
#allow established communication
-A INPUT -p tcp -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p udp -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p ipv6-icmp -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

#allow ping
-A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 128 -m conntrack --ctstate NEW -j ACCEPT
#allow public services
-A INPUT -p tcp -m tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT
#check portknocking and allow protected services
-A INPUT ! -s ::1 -j PORTKNOCK
-A INPUT -p tcp -m tcp -m multiport --dports 22 -m conntrack --ctstate NEW -m recent --rcheck --seconds 60 --name PORTKNOCK --reap -j ACCEPT

#allow outgoing communication
-A OUTPUT -o lo -j ACCEPT
-A OUTPUT -p tcp -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -p udp -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -p ipv6-icmp -j ACCEPT
COMMIT
```

# How to use with ssh client

You can configure your ssh client to automaticaly knock to the server prior connecting, install netcat for this.
Create following configuration in your .ssh/config

```text
# ~/.ssh/config
Host server
	HostName server.example.com
	#add "-4" to nc params to have IPv4 connection to the dualstack system
	ProxyCommand sh -c "nc -z -w1 %h 8881 ; nc -z -w1 %h 7777; nc -z -w1 %h 9991; nc -z -w1 %h 9992; nc %h %p"
```

and then just ssh to it

```bash
$ ssh -v server
...
debug1: Executing proxy command: exec sh -c "nc -z -w1 server.example.com 8881 ; nc -z -w1 server.example.com 7777; nc -z -w1 server.example.com 9991; nc -z -w1 server.example.com 9992; nc server.example.com 22"
...
server.example.com ~ $
```

# How to see the ip lists

if you want to see the lists of iptables module `recent`, just cat /proc/net/xt_recent/{name used in recent module}

# See too

- [w00tw00t.at.ISC.SANS.DFind](http://timkunze.eu/w00tw00t-at-isc-sans-dfind/)
- [ArchWiki Port Knocking](https://wiki.archlinux.org/index.php/Port_Knocking)
- [Multiple-port knocking Netfilter/IPtables only implementation](http://www.debian-administration.org/articles/268)
- [How To Configure Port Knocking Using Only IPTables on an Ubuntu VPS](https://www.digitalocean.com/community/articles/how-to-configure-port-knocking-using-only-iptables-on-an-ubuntu-vps)
