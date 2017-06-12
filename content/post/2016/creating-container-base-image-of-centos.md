+++
date = "2016-07-01T06:30:00+02:00"
tags = [ "rkt", "centos", "docker" ]
title = "Creating container base image of CentOS"
+++

[Docker docs] and [ACI docs] decsribe the steps how to create base images from existing tarballs/folders with root file systems of distribution. If you make a deeper look, you will probably [find the CentOS tarballs] which [are used by docker] for creation of centos base images.

But how to get this root file system tree? This blogpost covers the creation of this root file system tree for CentOS and the creation of base images for Docker and Rkt.

<!--more-->

# CentOS root file system tree

You need existing CentOS 7 system for this. You can use a [vagrant environment] if you want. Besides that, you have to exectute the steps below as root.

```bash
# Create a folder for our new root structure
$ export centos_root='/centos_image/rootfs'
$ mkdir -p $centos_root
# initialize rpm database
$ rpm --root $centos_root --initdb
# download and install the centos-release package, it contains our repository sources
$ yum reinstall --downloadonly --downloaddir . centos-release
$ rpm --root $centos_root -ivh centos-release*.rpm
$ rpm --root $centos_root --import  $centos_root/etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
# install yum without docs and install only the english language files during the process
$ yum -y --installroot=$centos_root --setopt=tsflags='nodocs' --setopt=override_install_langs=en_US.utf8 install yum
# configure yum to avoid installing of docs and other language files than english generally
$ sed -i "/distroverpkg=centos-release/a override_install_langs=en_US.utf8\ntsflags=nodocs" $centos_root/etc/yum.conf
# chroot to the environment and install some additional tools
$ cp /etc/resolv.conf $centos_root/etc
$ chroot $centos_root /bin/bash <<EOF
yum install -y procps-ng iputils
yum clean all
EOF
$ rm -f $centos_root/etc/resolv.conf
```

# Docker base image

```bash
# install and enable docker
$ yum install -y docker
...
$ systemctl start docker
# create docker image
$ tar -C $centos_root -c . | docker import - centos
sha256:28843acfa85226385d2db0196db72465729db38088ea98751d4a5c3d4c85ccd
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              28843acfa852        11 seconds ago      287.7 MB
# run docker image
$ docker run --rm centos cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)
```

Via usual docker ways you can now export the image or push it to some registry.

# ACI image for rkt

You will nedd [actool] to build an ACI image from root tree. You can build it by yourself or [get it from my OBS repository]:
```bash
$ wget http://download.opensuse.org/repositories/home:artem_sidorenko:rkt/CentOS_7/home:artem_sidorenko:rkt.repo \
  -O /etc/yum.repos.d/rkt.repo
$ yum install -y actool
...
# its a good idea to install rkt from same repository to verify the image
$ yum install -y rkt
...
```

Rkt uses [ACI][aci description] images and they require a [manifest file][aci manifest], which describes the image.

```bash
$ export centos_manifest='/centos_image/manifest'
$ cat > $centos_manifest <<EOF
{
    "acKind": "ImageManifest",
    "acVersion": "0.0.1",
    "name": "centos",
    "labels": [
        {"name": "os", "value": "linux"},
        {"name": "arch", "value": "amd64"},
        {"name": "version", "value": "0.0.1"}
    ],
    "app": {
        "exec": [
            "/bin/bash"
        ],
        "user": "0",
        "group": "0"
    }
}
EOF
$ actool build /centos_image centos.aci
# import the image to rkt
$ rkt fetch --insecure-options=image centos.aci
$ rkt image list
ID			NAME	IMPORT TIME	LAST USED	SIZE	LATEST
sha512-a8d4915265a0	centos	16 seconds ago	15 seconds ago	558MiB	false
# run the image with rkt
$ rkt run centos --exec cat -- /etc/redhat-release ---
image: using image from local store for image name coreos.com/rkt/stage1-coreos:1.9.1
image: using image from local store for image name centos
networking: loading networks from /etc/rkt/net.d
networking: loading network default with type ptp
[ 4135.403735] centos[5]: CentOS Linux release 7.2.1511 (Core)
```

# See too

- [Create CentOS base Docker image for PowerPC platform](http://cloudgeekz.com/752/centos-docker-image-power.html)
- [Building ACIs][ACI docs]

[Docker docs]: https://docs.docker.com/engine/userguide/eng-image/baseimages/
[ACI docs]: https://github.com/appc/spec#building-acis
[find the centos tarballs]: https://github.com/CentOS/sig-cloud-instance-images/tree/CentOS-7.2.1511/docker
[are used by docker]: https://github.com/docker-library/official-images/blob/master/library/centos
[vagrant environment]: https://gitlab.com/artem-sidorenko/vagrant-environments
[aci description]: https://coreos.com/rkt/docs/latest/app-container.html#aci
[aci manifest]: https://github.com/appc/spec/blob/master/spec/aci.md#image-manifest
[actool]: https://github.com/appc/spec/tree/master/actool
[get it from my OBS repository]: https://software.opensuse.org/download.html?project=home%3Aartem_sidorenko%3Arkt&package=actool
