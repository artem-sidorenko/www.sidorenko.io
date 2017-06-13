+++
date = "2011-04-25T21:15:00+01:00"
tags = [ "freebsd", "bind" ]
title = "Using bind from ports"
+++

There are some reasons to use bind from ports and not from world. One reason could be if you want to use [DLZ](http://bind-dlz.sourceforge.net/).

<!--more-->

# Configure world
We are going to remove bind from world but to keep the bind configs and scripts.

## src.conf

Create or change `/etc/src.conf` and place following vars in this file:

```
WITHOUT_BIND_DNSSEC="YES"
WITHOUT_BIND_LIBS_LWRES="YES"
WITHOUT_BIND_NAMED="YES"
WITHOUT_BIND_UTILS="YES"
```

Details of this configuration are documented in man src.conf

## Rebuild and reinstall world

```bash
$ cd /usr/src
$ make buildworld
$ make installworld
$ make -DBATCH_DELETE_OLD_FILES delete-old
$ make -DBATCH_DELETE_OLD_FILES delete-old-libs
```

If you are missing /usr/src or are unsure, read [this](http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/makeworld.html).

## Merge configs

```bash
$ cd /usr/src
$ mergemaster
```

# Build bind from ports

```bash
$ cd /usr/ports/dns/bind98
$ make config
```

Select "REPLACE_BASE" and start the build

```bash
$ make install clean
```

## Start bind

Switch it on and start it

```bash
$ echo 'named_enable="YES"'>>/etc/rc.conf
$ /etc/rc.d/named start
```
