+++
date = "2016-05-25T19:15:29+02:00"
tags = [ "deb", "linux", "packaging", "ubuntu", "debian" ]
title = "A short way to a deb package"
+++

I'm playing with [coreos rkt], and I was missing [rkt DEB packages] for Ubuntu systems.
CoreOS rkt provides [the tgz archives][rkt tgz archives] with compiled software. My idea was just to package this archives to DEBs in order to get easy distribution or updates of rkt on my systems.

The easy way is to use [fpm] for this, but I wanted to use [OpenBuild Service of OpenSuse][openbuild service] in order to build RPMs and DEBs in the same time (this is covered in the [next blogpost](/blog/2016/05/26/build-packages-with-openbuild-service/)). This was the main reason to go more or less the Debian packaging way.

Debian packaging way is powerful, really. On the other hand, this power and the amount of possible solution ways are a bit confusing for beginners: actually you have to read the big amount of [debian packaging resources] in order to get a picture about all existing use cases, different tools and sometimes different information sources you need.

My situation was quite similar: I was missing a guide or some tutorial for my simple use case and I didn't want to invest so much time for a simple "repackaging" from tgz to deb, but I had to. This blog post provides a such tutorial, based on my simple use case. But keep in mind, this short post doesn't replace the [debian packaging resources] like [Debian New Maintainers Guide](https://www.debian.org/doc/manuals/maint-guide/index.en.html) or [Debian Policy Manual](https://www.debian.org/doc/debian-policy/) and has low quality claim then usual packages provided by distributions.

<!--more-->

The main goal is to take the [rkt tgz archive][rkt tgz archives] and to create a DEB package, where the rkt files are placed on the right places. I'm doing this on the blank ubuntu 14.04 environment (I usually use a docker container with my [vagrant dev environment setup][my vagrant envs]).

# Environment setup

```bash
# install packaging tools, dh-make installation will include
# all other required packages as dependencies
$ sudo apt-get install dh-make
# create the folder, where you will do your packaging activities
$ mkdir rkt-packaging
$ cd rkt-packaging
# download rkt tgz file
$ wget https://github.com/coreos/rkt/releases/download/v1.6.0/rkt-v1.6.0.tar.gz
# unpack it and remove the tarball
$ tar xfz rkt-v1.6.0.tar.gz && rm rkt-v1.6.0.tar.gz
# rename the directory to be compliant with debian naming
$ mv rkt-v1.6.0 rkt-1.6.0
# change to the directory
$ cd rkt-1.6.0
```

# Packaging

You can use `dh_make` (its a helper for creation of deb packages from source tarballs): it would create a debian folder with a lot of files, which you can update as described below. In order to give more focus and explaination, we will not use `dh_make` but create all files manually.

## Create the debian subfolder

This folder contains all required information about the package and instructions how to build it. Somewhere I saw even the recommendation and wish from Debian package maintainers to the upstream projects: if you want to simplify the life of package maintainers, include this debian folder within your distribution tarballs or version control. [This page][debian directory required files] provides an overview over required files in this directory. We will create them in the steps below.

```bash
# create a magic debian directory
$ mkdir debian
```

## Create the control file

This file providers the package metadata. Tools like dpkg or apt are using this file to get the information about the package.

Create `debian/control` with a content like below. You can find more information on the control fields [here](https://www.debian.org/doc/debian-policy/ch-controlfields.html).

```text
# debian/control
Source: rkt
Section: admin
Priority: optional
Maintainer: Artem Sidorenko <artem@posteo.de>
Build-Depends: debhelper (>= 8.0.0)
Standards-Version: 3.9.4
Homepage: https://github.com/coreos/rkt

Package: rkt
Architecture: amd64
Depends: ${shlibs:Depends}
Description: A security-minded, standards-based container engine
 rkt is the next-generation container manager for Linux clusters. Designed for security, simplicity, and composability within modern cluster architectures, rkt discovers, verifies, fetches, and executes application containers with pluggable isolation.
```

## Create compat file

The compat file defines the debhelper (debhelper will be explained later) compability level. Just set `debian/compat` [to 9](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#compat)

```text
# debian/compat
9
```

## Create rules file

Its the most interesting part. The `debian/rules` file contains the instructions how to compile, build and package the software. The rules file is a [Makefile](https://www.gnu.org/software/make/manual/html_node/index.html). Tools like [debhelper] are there in order to simplify the life of package maintainers. Debhelper provides a lot of functionality and covers the usual scenarious via different Makefile targets. In case of differences or some special cases for particilar packages, you can extend the procedure steps of debhelper or override them.

After the usual build phase (which we don't have in this case), the software [should be](https://pkg-perl.alioth.debian.org/debhelper.html#note_on_paths) 'installed' to `debian/[package-name]`. In the simple view this can be seen as a root folder of your future package.

In our case we want just to copy some files to the right locations, so we will override the `dh_auto_install` (this is done via a target with `override_` prefix).

Create the `debian/rules` with content like below. **Be carefull, this is a Makefile, so the indents are tabs and not spaces!**

```makefile
#!/usr/bin/make -f
# debian/rules
# -*- makefile -*-

export DESTROOT=$(CURDIR)/debian/rkt

%:
	dh $@

override_dh_auto_install:
	dh_auto_install
	install -p -D -m 0755 rkt $(DESTROOT)/usr/sbin/rkt
	install -d $(DESTROOT)/usr/lib/rkt
	install -p -m 0644 stage1-* $(DESTROOT)/usr/lib/rkt/
```

## Create changelog file

This file is a changelog of the package with a [defined format][changelog format].

Create `debian/changelog` with a content like below:

```text
# debian/changelog
rkt (1.6.0-0) stable; urgency=low

  * Release 1.6.0

 -- Artem Sidorenko <artem@posteo.de>  Thu, 23 May 2016 10:00:00 +0100
```

## Build the package

```bash
# build the package
$ dpkg-buildpackage -B
...
dpkg-deb: building package `rkt' in `../rkt_1.6.0-0_amd64.deb'.
...
```

# See too

* [Debian Packaging Tutorial](https://www.debian.org/doc/manuals/packaging-tutorial/packaging-tutorial.en.pdf) (the most shortest and in the same a pretty complete overview I saw)
* [Debian packaging resourrces][debian packaging resources]
* [Debhelper cheatsheet][debhelper]
* [Required files in debian directory][debian directory required files]

[coreos rkt]: https://github.com/coreos/rkt
[rkt DEB packages]: https://github.com/coreos/rkt/issues/867
[rkt tgz archives]: https://github.com/coreos/rkt/releases
[fpm]: https://github.com/jordansissel/fpm
[openbuild service]: https://build.opensuse.org/
[debian packaging resources]: https://wiki.debian.org/Packaging
[my vagrant envs]: https://github.com/artem-sidorenko/vagrant-environments
[debhelper]: https://pkg-perl.alioth.debian.org/debhelper.html
[changelog format]: https://www.debian.org/doc/debian-policy/ch-source.html#s-dpkgchangelog
[debian directory required files]: https://www.debian.org/doc/manuals/maint-guide/dreq.en.html
