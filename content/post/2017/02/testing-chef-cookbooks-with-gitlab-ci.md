+++
date = "2017-02-11T22:06:03+01:00"
tags = [ "chef", "gitlab", "ci" ]
title = "Testing chef cookbooks with GitLab CI"
+++

[Test Kitchen] is a common tool for integration testing of [Chef cookbooks]. Usually a combination of [Vagrant]&[VirtualBox] is used to bring up the VMs. This works well for local development setups, but what about Continuous Integration environments? You can find several approaches how cookbooks can be tested in the CI:

- if your CI runs on non-persistent VMs (like [Travis CI]) you might use [kitchen-dokken] like [chef-ssh-hardening] does
- you can use kitchen providers like [kitchen-digitalocean] or [kitchen-ec2]

Well, but what about the case you want to use Vagrant&VirtualBox in the CI too? There are some reasons for this approach:

- Maybe you can not use public cloud providers for some reasons and do not have your own on-premise cloud like OpenStack
- Maybe you want to use the same setup/technologies in the CI and locally as you want be able to easy reproduce errors and problems

[GitLab] is quite often used in the enterprise environments, where restrictions on the public cloud usage may apply. GitLab has its own [GitLab CI], which can be easily used for cookbook testing.

This post covers a basic GitLab CI setup with Test Kitchen and Vagrant&VirtualBox as backend.

<!--more-->

We will setup a physical host (VirtualBox [doesn't support nested virtualization](https://www.virtualbox.org/ticket/4032)) with CentOS 7 (Ubuntu setup should be similar):

- VirtualBox package will be installed on the host in order to have the kernel modules available
- CI jobs will be executed in the Docker containers in order to have a clean environment for each job
- Docker image will have VirtualBox&Vagrant and [ChefDK] installed

We will also setup an additional runner with [docker-in-docker], which will be used with [GitLab Registry].

We assume following things:

- your GitLab address is `https://gitlab.example.com`
- your GitLab Registry address is `registry.example.com`

# Some words on caching and speed optimizations

There are some methods which can significantly speedup the CI test cycles:

1. caching of vagrant boxes via persistent cache folder
1. caching of chef packages (native support starting from [test-kitchen 1.14.0](https://github.com/test-kitchen/test-kitchen/blob/master/CHANGELOG.md#v1140-2016-11-22), [kitchen-vagrant 0.21.0](https://github.com/test-kitchen/kitchen-vagrant/blob/master/CHANGELOG.md#0210-2016-11-29), [chefdk 1.0.3](https://github.com/chef/chef-dk/commit/51aa5895d7419427972456a1e69a7111801098f2))
1. caching of OS packages via [vagrant-cachier]
1. usage of [VirtualBox linked clones](https://www.hashicorp.com/blog/vagrant-1-8.html)

All this methods work without issues only if your runners behave like a usual installation on the development system.
The main reason is the conflict between different processes trying to access the same resource (e.g. update of same apt cache in different VMs).

In the end:

- its a good idea to have persistent cache folders for your runners
- its a good idea to avoid multiple concurrent jobs per runner, but to have multiple runners where each runner handles only one job concurrently

# Basic setup of CI host

Setup the repositories:

```bash
$ cat > /etc/yum.repos.d/virtualbox.repo <<EOF
[virtualbox]
name=Oracle Linux / RHEL / CentOS-$releasever / $basearch - VirtualBox
baseurl=http://download.virtualbox.org/virtualbox/rpm/el/$releasever/$basearch
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://www.virtualbox.org/download/oracle_vbox.asc
EOF
$ cat > /etc/yum.repos.d/docker.repo <<EOF
[docker]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=0
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
$ cat > /etc/yum.repos.d/gitlab_runner.repo <<EOF
[gitlab_runner]
name=GitLab Runner Repository
baseurl=https://packages.gitlab.com/runner/gitlab-ci-multi-runner/el/7/$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/runner/gitlab-ci-multi-runner/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOF
```

Repositories are disabled per default for the case to avoid uncontrolled package updates.

Install software:

```bash
$ yum -y --enablerepo=virtualbox --enablerepo=docker --enablerepo=gitlab_runner \
      install VirtualBox-5.1 docker-engine gitlab-runner
```

Allow gitlab-runner to talk to the docker daemon and start the services:

```bash
$ usermod -G docker -a gitlab-runner
$ systemctl restart gitlab-runner
$ systemctl start docker
$ systemctl enable gitlab-runner
$ systemctl enable docker
```

**Note:** my best experience with docker on RHEL 7 is with btrfs storage backend. However the detailed setup guide is out-of-scope for this blogpost, see [this document](https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/) for more information.

# Setting up the gitlab-runner for docker image building

Please enable [GitLab Registry] on your GitLab installation.

Register the runner on the CI host (please replace `[TOKEN]` with runner registration token you see in the GitLab interface):

```bash
$ gitlab-runner register -n -u https://gitlab.example.com -r [TOKEN] \
  --name dind --tag-list dind --executor docker --docker-image docker:latest \
  --docker-privileged --docker-services docker:dind \
  --docker-allowed-services docker:dind \
  --docker-allowed-images docker:latest --docker-allowed-images docker:dind
```

See [this page][Using Docker Build] for further information.

# Docker image for Chef test jobs

Create a new repository, ensure 'Builds' is enabled in the project settings and the dind-runner can be used by the new project.

We assume that repository/project name is `docker/docker-test-kitchen`.

Add following CI settings to the repository, create `.gitlab-ci.yml`:

```yaml
# .gitlab-ci.yml
stages:
- build

variables:
  CONTAINER_TEST_IMAGE: registry.example.com/$CI_PROJECT_PATH:$CI_BUILD_REF_NAME
  CONTAINER_RELEASE_IMAGE: registry.example.com/$CI_PROJECT_PATH:latest

build:
  stage: build
  script:
   - docker build -t $CONTAINER_TEST_IMAGE .
  except:
   - master
  tags:
   - dind

build and release:
  stage: build
  script:
   - docker build -t $CONTAINER_RELEASE_IMAGE .
   - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN [REGISTRY ADDRESS]
   - docker push $CONTAINER_RELEASE_IMAGE
  only:
   - master
  tags:
   - dind
```

Add following `Dockerfile` to the repository:

```plain
# Dockerfile
FROM centos:7
MAINTAINER Artem Sidorenko <artem@posteo.de>

# Software versions
ENV VIRTUALBOX_VERSION 5.1
ENV CHEFDK_VERSION 1.2.22
ENV VAGRANT_VERSION 1.9.1

# Add some asset files to the image
ADD assets /

# Configure software repositories
RUN rpm --import https://www.virtualbox.org/download/oracle_vbox.asc && \
    curl -o /etc/yum.repos.d/virtualbox.repo http://download.virtualbox.org/virtualbox/rpm/el/virtualbox.repo && \
    rpm --import https://packages.chef.io/chef.asc && \
    echo $'[chef-stable]\n\
name=chef-stable\n\
baseurl=https://packages.chef.io/repos/yum/stable/el/7/\$basearch/\n\
gpgcheck=1\n\
enabled=1\n'\
    >> /etc/yum.repos.d/chef.repo

RUN yum -y install "VirtualBox-$VIRTUALBOX_VERSION" "chefdk-$CHEFDK_VERSION" && \
    yum -y install "https://releases.hashicorp.com/vagrant/${VAGRANT_VERSION}/vagrant_${VAGRANT_VERSION}_x86_64.rpm" && \
    yum clean all && \
    vagrant plugin install vagrant-cachier
```

Add following Vagrantfile to the repo, create `assets/root/.vagrant.d/Vagrantfile`:

```ruby
# assets/root/.vagrant.d/Vagrantfile
Vagrant.configure('2') do |config|
  # enable caching of packages via vagrant-cachier
  config.cache.scope = :box
  # enable linked clones in order to speedup the boot process
  config.vm.provider 'virtualbox' do |v|
    v.linked_clone = true
  end
end
```

Then commit and push it to GitLab, GitLab CI should build and push the image to the GitLab Registry.

# Setting up the runner for Chef testing

When CI pushed the image to the registry, you can register the new runner on the CI host (please replace `[TOKEN]` with runner registration token you see in the GitLab interface):

```bash
$ gitlab-runner register -n -u https://gitlab.example.com -r [TOKEN] \
  --name test-kitchen --tag-list test-kitchen --executor docker --limit 1 \
  --docker-image registry.example.com/docker/docker-test-kitchen:latest --docker-privileged \
  --docker-volumes '/home/gitlab-runner/test-kitchen/vagrant-boxes:/root/.vagrant.d/boxes:rw' \
  --docker-volumes '/home/gitlab-runner/test-kitchen/virtualbox-vms:/root/VirtualBox VMs:rw' \
  --docker-volumes '/home/gitlab-runner/test-kitchen/virtualbox-config:/root/.config/VirtualBox:rw' \
  --docker-volumes '/home/gitlab-runner/test-kitchen/vagrant-cachier:/root/.vagrant.d/cache:rw' \
  --docker-volumes '/home/gitlab-runner/test-kitchen/kitchen-cache:/root/.kitchen/cache:rw' \
  --docker-allowed-services '' \
  --docker-allowed-images registry.example.com/docker/docker-test-kitchen:latest
```

As you noticed, we configured the runner with concurrent job limit 1 and added some persistent folders:

- persistent folders for caches
- persistent folders for VirtualBox data: we need this because of linked clones: a linked clone is a VM which should survive over different CI jobs

In a production setup you might want to setup more runners with a same tag in order to server multiple jobs concurrently.

# Configuring the cookbook repository

Create a new repository, ensure `Builds` is enabled in the project settings and the test-kitchen-runner can be used by the new project.

Create a simple cookbook:

```bash
$ cat > metadata.rb <<EOF
name 'cookbook'
maintainer 'Artem Sidorenko'
maintainer_email 'artem@example.com'
license 'all_rights'
description 'Demo cookbook'
long_description 'Demo cookbook'
version '0.0.1'
source_url 'https://gitlab.example.com/chef/cookbook'
issues_url 'https://gitlab.example.com/chef/cookbook/issues'
EOF
$ mkdir -p recipes test/integration/default
$ echo "packages 'httpd'" > recipes/default.rb
$ cat > test/integration/default/test_spec.rb <<EOF
describe package('httpd') do
  it { should be_installed }
end
EOF
$ cat > .kitchen.yml <<EOF
---
driver:
  name: vagrant

provisioner:
  name: chef_zero

verifier:
  name: inspec

platforms:
  - name: centos-7
    driver_config:
      box: bento/centos-7.3

suites:
  - name: default
    run_list:
      - recipe[cookbook]
EOF
```

Create GitLab CI configuration, create `.gitlab-ci.yml`:

```yaml
# .gitlab-ci.yml
test-kitchen:
  stage: test
  artifacts:
    expire_in: 7d
    when: on_failure
    paths:
      - .kitchen/logs
  script:
   - kitchen test -d always -c 3
  tags:
    - test-kitchen
```

Commit and push it to GitLab, GitLab CI should run the kitchen on your gitlab-runner.

[Chef cookbooks]: https://docs.chef.io/cookbooks.html
[Test Kitchen]: http://kitchen.ci/
[kitchen-dokken]: https://github.com/someara/kitchen-dokken
[chef-ssh-hardening]: https://github.com/dev-sec/chef-ssh-hardening
[Travis CI]: https://travis-ci.org/
[kitchen-digitalocean]: https://github.com/test-kitchen/kitchen-digitalocean
[kitchen-ec2]: https://github.com/test-kitchen/kitchen-ec2
[GitLab]: https://about.gitlab.com/
[GitLab CI]: https://docs.gitlab.com/ce/ci/
[Vagrant]: https://www.vagrantup.com/
[VirtualBox]: https://www.virtualbox.org/
[ChefDK]: https://downloads.chef.io/chefdk
[docker-in-docker]: https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
[GitLab Registry]: https://about.gitlab.com/2016/05/23/gitlab-container-registry/
[Using Docker Build]: https://docs.gitlab.com/ce/ci/docker/using_docker_build.html
[vagrant-cachier]: https://github.com/fgrehm/vagrant-cachier
