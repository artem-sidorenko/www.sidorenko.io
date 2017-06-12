+++
date = "2010-10-21T22:09:09+01:00"
tags = [ "linux", "cisco", "network" ]
title = "How to determine the switch port with tcpdump"
+++

We are going to use the Cisco Discovery Protocol, CDP

<!--more-->

```bash
linux$ tcpdump -i eth0 ether host 01:00:0c:cc:cc:cc -s 1500 -vv -nn
```

and wait a minute..

See also:

- [Determine Port a Server is connected to](http://www.velocityreviews.com/forums/showpost.php?p=198614&postcount=4)
- [tcpdump filter for capturing only Cisco Discovery Protocol (CDP) Packets](http://sidewynder.blogspot.com/2005/07/tcpdump-filter-for-capturing-only.html)
