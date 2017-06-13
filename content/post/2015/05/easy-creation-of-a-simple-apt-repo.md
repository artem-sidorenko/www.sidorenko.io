+++
date = "2015-05-19T16:12:51+02:00"
tags = [ "apt", "centos", "redhat", "ubuntu", "dpkg", "deb", "debian" ]
title = "Easy creation of a simple apt repo"
+++

If wanted to setup a very simple apt repository on RHEL system to server a small environment with Debian/Mint systems. So the goal is just to place some already built deb packages to some apt repository. Unfortunally it turned to be not a straight forward task like with yum repos. All common ways are based on the Debian/Ubuntu scripts, which are not available on the RHEL platform via packages.

This post covers following:

- Creation of APT repository on very simple way without involvment of usual tools like apt-ftparchive or [reporepo](https://mirrorer.alioth.debian.org/)
- APT repository will be GPG signed, to allow verification of integrity
- APT repository will be build on the CentOS/RHEL platform and serve Ubuntu/Debian clients

<!--more-->

# Apt repository on the server

- Create or import gpg keys you want to use for signing
- Create a new folder `apt-repo` and place your deb packages to this folder
- Make this folder available via http/https

```bash
#install epel as we need dpkg from epel-testing
$ yum -y install epel-release
#install dpkg binaries from epel-testing
$ yum -y install --enablerepo=epel-testing dpkg-dev tar

#lets create the package index
$ cd apt-repo
$ dpkg-scanpackages -m . > Packages
$ cat Packages | gzip -9c > Packages.gz

#create the Release file
$ PKGS=$(wc -c Packages)
$ PKGS_GZ=$(wc -c Packages.gz)
$ cat > Release << EOF
Architectures: all
Date: $(date -R)
MD5Sum:
 $(md5sum Packages  | cut -d" " -f1) $PKGS
 $(md5sum Packages.gz  | cut -d" " -f1) $PKGS_GZ
SHA1:
 $(sha1sum Packages  | cut -d" " -f1) $PKGS
 $(sha1sum Packages.gz  | cut -d" " -f1) $PKGS_GZ
SHA256:
 $(sha256sum Packages | cut -d" " -f1) $PKGS
 $(sha256sum Packages.gz | cut -d" " -f1) $PKGS_GZ
EOF

#and sign it
$ gpg -abs -o Release.gpg Release
```

# Configure the clients

```bash
#add the public key to the apt keyring
$ apt-key add public-key.file
#add the new repo
$ echo "deb http://some-url/apt-repo/ / " > /etc/apt/sources.list.d/new-repo.list
#and update
$ apt-get update
```

# See too

- [How to setup a Debian repository](https://wiki.debian.org/HowToSetupADebianRepository)
- [Repository Format](https://wiki.debian.org/RepositoryFormat)
