+++
date = "2012-03-07T12:50:00+01:00"
tags = [ "linux", "cluster", "drbd" ]
title = "DRBD recovery from split brain"
+++

If you get degraded DRBD with log messages like "Split-Brain detected but unresolved, dropping connection!", you have to manually resolve split brain situation.

<!--more-->

# Possible reason for such situation

- You switch all cluster nodes in standby via cluster suite (i.e. pacemaker) at the same time, so you don't have any active drbd instance running.
- You switch the node, which was in Secondary status in drbd last time, online.
- You switch other nodes online

Result:

```bash
#/proc/drbd on the node1
version: 8.4.0 (api:1/proto:86-100)
GIT-hash: 28753f559ab51b549d16bcf487fe625d5919c49c build by gardner@, 2011-12-1
2 23:52:00
 0: cs:StandAlone ro:Secondary/Unknown ds:UpToDate/DUnknown   r-----
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:76

#log message on the node1
Mar  7 15:38:05 node1 kernel: block drbd0: Split-Brain detected but unresolved, dropping connection!

#/proc/drbd on the node2
version: 8.4.0 (api:1/proto:86-100)
GIT-hash: 28753f559ab51b549d16bcf487fe625d5919c49c build by gardner@, 2011-12-1
2 23:52:00
 0: cs:StandAlone ro:Primary/Unknown ds:UpToDate/DUnknown   r-----
    ns:0 nr:0 dw:144 dr:4205 al:5 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:100
```

# Manually resolve this split brain situation

Start drbd manually on both nodes

Define one node as secondary and discard data on this

```bash
drbdadm secondary all
drbdadm disconnect all
drbdadm -- --discard-my-data connect all
```

**Think about the right node to discard the data, otherwise you can lose some data**

Define another node as primary and connect

```bash
drbdadm primary all
drbdadm disconnect all
drbdadm connect all
```

# Configure drbd for auto split brain resolving

Place this configuration in `drbd.conf` on both nodes

```text
net {
  after-sb-0pri discard-zero-changes;
  after-sb-1pri discard-secondary;
  after-sb-2pri disconnect;
}
```

# See too

- [Manual split brain recovery](http://www.drbd.org/users-guide/s-resolve-split-brain.html)
- [Configuring split brain behavior](http://www.drbd.org/users-guide/s-configure-split-brain-behavior.html)
