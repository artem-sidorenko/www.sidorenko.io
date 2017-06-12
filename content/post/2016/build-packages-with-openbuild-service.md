+++
date = "2016-05-26T11:40:11+02:00"
tags = [ "linux", "packaging", "rpm", "deb", "obs" ]
title = "Build packages with OpenBuild service"
+++

As I already mentioned in the [previous][deb blogpost] blog post, initially I wanted to use [OpenBuild Service from openSUSE (OBS)][openbuild service] to build packages of [CoreOS rkt][coreos rkt]. OBS allows you to build packages for different platforms, e.g. RPMs for CentOS, RH, Fedora, OpenSUSE and in the same time DEBs for Ubuntu, Debian. Another positive thing: OpenBuild Service provides yum and apt repositories, which allow easy distribution and updates of packages.

CoreOS rkt provides [the tgz archives][rkt tgz archives] with compiled software. The idea is to package this archives to RPMs/DEBs with help of OBS.

This blog post covers the required steps in order to achieve this goal with this simple use case. However you should keep in mind, the packaging instructions (e.g. spec or rules files) are representing a simple example only: they have low quiality claim compared to the distributions.

<!--more-->

# Create project on OpenBuild service

- First of all create an account at [https://build.opensuse.org/](https://build.opensuse.org/)
- OBS is organized in projects, after login you will see your home project. This home project can contain packages or even have subprojects for better organization.
- Lets create a subproject via Web interface:
  - Go to your home project
  - Switch to `Subprojects`
  - `New subproject` - Subproject name: `test`
- Configure the target package distributions:
  - Switch to `Repositories` in your new project
  - Select `Add repositories` and choose CentOS 7 and Ubuntu 16.04
  - Now you can configure the architectures, please disable the `i586` as our package will be only for `x86_64`

# Create a package

[Osc tool][osc tool] is a CLI/API interface for OpenBuild Service. It's very similar to the subversion CLI interface. We will use osc for package creation and upload of source files.

- Install osc tool, e.g. with `apt-get install osc` on Debian/Ubuntu
- Start osc and provide the credentials, osc will create `~/.oscrc` file with settings:

```bash
$ osc

Your user account / password are not configured yet.
You will be asked for them below, and they will be stored in
/home/vagrant/.oscrc for future use.

Creating osc configuration file /home/vagrant/.oscrc ...
Username: artem_sidorenko
Password:
done
Usage: osc [GLOBALOPTS] SUBCOMMAND [OPTS] [ARGS...]
or: osc help SUBCOMMAND

openSUSE build service command-line tool.
...
```

- Now create a new package in the `test` project, lets use `rkt` as name

```bash
$ osc meta pkg -e home:artem_sidorenko:test rkt
# fill the data
```

- Checkout the project

```bash
$ osc checkout home:artem_sidorenko:test
A    home:artem_sidorenko:test
A    home:artem_sidorenko:test/rkt
At revision None.
# change to the package directory in the project
$ cd home:artem_sidorenko:test/rkt
```

Now we start adding files to the `home:artem_sidorenko:test/rkt` and let OpenBuild Service build the packages

# Download sources

Download the [rkt binary tgz][rkt tgz archives]:

```bash
# download the rkt tgz with binaries
$ wget https://github.com/coreos/rkt/releases/download/v1.6.0/rkt-v1.6.0.tar.gz
...
```

# Building RPM

RPM building is pretty simple, you need the source files and the spec file. Spec file contains all steps for package creation. In our simple case it just copies the files to the right places. Create `rkt.spec` with a following content:

```text
# rkt.spec
Name:           rkt
Version:        1.6.0
Release:        0
License:        Apache-2.0
Summary:        A security-minded, standards-based container engine
Url:            https://coreos.com/rkt
Source0:        %{name}-v%{version}.tar.gz

%description
rkt is the next-generation container manager for Linux clusters. Designed for security, simplicity, and composability within modern cluster architectures, rkt discovers, verifies, fetches, and executes application containers with pluggable isolation.

%prep
%setup -q -n %{name}-v%{version}

%install
# dirs
install -dp %{buildroot}/%{_bindir}
install -dp %{buildroot}/%{_libexecdir}/%{name}
install -dp %{buildroot}/%{_sharedstatedir}/%{name}

# binaries and stages
install -p -m 755 %{name} %{buildroot}%{_bindir}
install -p -m 644 stage1-*.aci %{buildroot}%{_libexecdir}/%{name}

%files
%defattr(-,root,root,-)
%{_bindir}/%{name}
%{_libexecdir}/%{name}/stage1-*.aci
```

Now lets upload the sources and spec file to OBS:

```bash
$ osc status
?    rkt-v1.6.0.tar.gz
?    rkt.spec
$ osc add rkt-v1.6.0.tar.gz rkt.spec
A    rkt-v1.6.0.tar.gz
A    rkt.spec
$ osc commit -m "rkt v1.6.0 rpm"
Sending    rkt-v1.6.0.tar.gz
Sending    rkt.spec
Transmitting file data .
Committed revision 1.
```

Go to the homepage of your test project and take a look to the build results, you should see status "building" for CentOS platform

# Building DEB

DEB building needs a bit more work and files. Please have a look to the [previous blog post][deb blogpost] for details. Following debian files are normally placed in the `debian` subfolder:

- `control` - package metadata information
- `rules` - build instructions
- `compat` - debhelper compatibility level - '9'
- `changelog`

When using OBS, you don't have to create the `debian` folder: just name with files with a debian prefix, e.g. `debian.control` for control file. OBS will take care about proper file relocation and naming prior to execution of Debian package commands.

Please create following files:

```text
# debian.control
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

```makefile
#!/usr/bin/make -f
# debian.rules
# Be carefull, this is a Makefile, so the indents are tabs and not spaces!
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

```text
# debian.compat
9
```

```text
# debian.changelog
rkt (1.6.0-0) stable; urgency=low

  * Release 1.6.0

 -- Artem Sidorenko <artem@posteo.de>  Thu, 26 May 2016 10:00:00 +0100
```

The next file we have to create, is a [debian source control file](https://wiki.debian.org/dsc) `.dsc`. OBS uses this file for generation of source package (and the binary package is built from this generated source package).

```text
# rkt.dsc
Format: 1.0
Source: rkt
Version: 1.6.0-0
Binary: rkt
Maintainer: Artem Sidorenko <artem@posteo.de>
Architecture: amd64
Homepage: https://github.com/coreos/rkt
Standards-Version: 3.9.4
Build-Depends: debhelper (>= 8.0.0)
```

Now lets upload the debian files to OBS:

```bash
$ osc status
?    debian.changelog
?    debian.compat
?    debian.control
?    debian.rules
?    rkt.dsc

$ osc add debian.* rkt.dsc
A    debian.changelog
A    debian.compat
A    debian.control
A    debian.rules
A    rkt.dsc
$ osc commit -m "rkt v1.6.0 deb"
Sending    debian.changelog
Sending    debian.compat
Sending    debian.control
Sending    debian.rules
Sending    rkt.dsc
Transmitting file data ..
Committed revision 2.
```

Go to the homepage of your test project and take a look to the build results, you should see status "building" for Ubuntu platform

# Download page with repositories

OBS offers fancy download pages for your projects or packages. This download pages provide instructions how to configure repositories for particilar distribution and how to install the software.

Such download page is addresed via a magic url:

```text
https://software.opensuse.org/download/package?project=[PROJECT]&package=[PACKAGE NAME]
```

`[PROJECT]` represents the project name, in this case it would be `home:artem_sidorenko:test`. Project yum/apt repositories offer all packages they have. The download page requires however `[PACKAGE NAME]` in order to provide installation instructions.

In our case the [download url](http://software.opensuse.org/download.html?project=home:artem_sidorenko:test&package=rkt) would be:

```text
http://software.opensuse.org/download.html?project=home:artem_sidorenko:test&package=rkt
```

(Hint: generation of this web page takes some time, so if you get 'not found' errors just try 10 minutes later again)

# Update procedure

Now to the most positive part: long term maintenance and updates:)

Given, rkt released version 1.7.0 and we want update the packages, the procedure would be like this:

- remove the old tarball rkt-v1.6.0.tar.gz
- add the new tarball rkt-v1.7.0.tar.gz
- update version information in `rkt.spec`
- update version information in `rkt.dsc`
- commit the changes and upload them to OBS

Easy?:)

# See too

- [openSUSE:Build Service Tutorial](https://en.opensuse.org/openSUSE:Build_Service_Tutorial)
- [Fedora Packaging Guidelines](https://fedoraproject.org/wiki/Packaging:Guidelines?-d=Packaging/Guidelines)
- [openSUSE:Build Service Debian builds]-https://en.opensuse.org/openSUSE:Build_Service_Debian_builds)
- [A Short Way to a Deb Package][deb blogpost]

[coreos rkt]: https://github.com/coreos/rkt
[rkt tgz archives]: https://github.com/coreos/rkt/releases
[openbuild service]: https://build.opensuse.org/
[osc tool]: https://en.opensuse.org/openSUSE:OSC
[deb blogpost]: /blog/2016/05/25/short-way-to-a-deb-package/
