+++
date = "2016-06-30T17:40:38+02:00"
tags = [ "chef", "centos", "raspberry pi" ]
title = "Installing chef on raspberry pi with centos"
+++

Chef is building omnibus packages only for x86. But probably you want to run chef on raspberry pi 3 with ARM.
There is a [blogpost][raspbian chef installation], which describes the chef installation on Raspbian. This blogpost covers the steps for chef installation on raspberry pi with centos.

<!--more-->

# Prepare the system

First of all, [install centos on your raspberry pi][centos installation].

Additionally I had to install ntpd in order to sync the clock:

```bash
$ yum install -y ntp
...
$ systemctl enable ntpdate ntpd
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpdate.service to /usr/lib/systemd/system/ntpdate.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
$ systemctl start ntpdate ntpd
...
```

# Install ruby

We install ruby from sources in order to have a modern ruby version, which is required by chef.

Install prerequisites for ruby compilation:

```bash
$ yum install -y gcc make zlib-devel openssl-devel
```

Install ruby 2.2 from sources:

```bash
$ mkdir /tmp/ruby
$ cd /tmp/ruby
$ curl -L --progress http://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.5.tar.gz | tar xz
...
$ cd ruby-2.2.5
$ ./configure --prefix=/opt/ruby \
              --disable-install-doc \
              --disable-install-rdoc \
              --disable-install-capi
...
$ make -j4
# grab a coffee, it takes a while
...
$ make install
...
$ cd
$ rm -rf /tmp/ruby
# add ruby binaries to the PATH
$ echo 'export PATH="$PATH:/opt/ruby/bin"' > /etc/profile.d/ruby.sh
$ chmod +x /etc/profile.d/ruby.sh
$ source /etc/profile
# disable docs for all gem installations to save storage place
$ echo "gem: --no-ri --no-rdoc --no-document" > ~/.gemrc
```

# Install chef

```bash
$ gem install chef
...
# run it:)
$ chef-client --version
Chef: 12.11.18
```

Ohai reports platform `derived` instead of `centos`:

```bash
$  ohai | grep platform
      "platform": "armv7l-linux-eabihf",
  "platform": "derived",
  "platform_version": null,
```

This would break the cookbooks, so lets fix it:

```bash
$ echo "CentOS Linux release 7.2.1511 (Core)" > /etc/redhat-release
$ ohai | grep platform
      "platform": "armv7l-linux-eabihf",
  "platform": "centos",
  "platform_version": "7.2.1511",
  "platform_family": "rhel",
```

# See too

- [Install Chef Client on a Raspberry Pi 2 Model B][raspbian chef installation]
- [CentOS Linux on the Raspberry Pi 3][centos installation]

[raspbian chef installation]: http://everydaytinker.com/raspberry-pi/installing-chef-client-on-a-raspberry-pi-2-model-b/
[centos installation]: https://wiki.centos.org/SpecialInterestGroup/AltArch/Arm32/RaspberryPi3
