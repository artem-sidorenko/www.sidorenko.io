+++
date = "2014-10-13T18:42:48+01:00"
tags = [ "puppet" ]
title = "Puppet structure"
+++

It not always easy to find a good folder hierarchy and structure for puppetmaster, as you have to find a way to combine it with different tools and workflows like git, [librarian][librarian] or [r10k][r10k].

As I still miss some kind of best practice whitepaper from puppetlabs, I want to cover here my view on this:

- with using hiera
- with or without using full autodeployment like [r10k][r10k]
- without any git submodules

<!--more-->

# Global structure

I don't want to mixup hiera and code, I see hiera only as a flexible way to get input values for the code.
I want to have some modules globally installed and active for all environments, and one folder with permissions only for according nodes, so we get following picture:

- /etc/puppet/
  - environments - to hold environment manifests, modules
  - hiera - to hold hiera information
  - modules - to hold global valid modules
  - secrets - to hold fqdn subfolders with some protected files

# Secrets

I don't want to have some files be available for all systems in my environment, but only to the target system itself. For example, some private keys: if you go with a usual modules concept and serve this files, they are available for all signed systems and can be fetched via HTTP. If an attacker gets access to one the puppet managed systems, all private keys would be compromised.
To avoid this we create the secrets folder and serve it subfolders via fileserver only to the according nodes

```ini
# /etc/puppet/fileserver.conf
[secrets]
path /etc/puppet/secrets/%H
#I had some issues with allow %H, so use *, your domain or ip range here
allow *
```

You can reference this with a ``puppet:///secrets/some_privatkey.pem`` and it will be sourced from ``/etc/puppet/secrets/some.fqdn/some_privatkey.pem``

If you want to get deeper and probably get the secrets auto generated for you, take a look at [puppet-secret][puppet-secret] from Dominik Richter.

# Hiera

With hiera we get a hierarchy with multiple sources for input data, which can be merged together. For details about hiera see [hiera introduction][puppet-hiera].
Our `/etc/puppet/hiera.yaml` looks like

```yaml
# /etc/puppet/hiera.yaml
---
:backends:
 - yaml
:yaml:
  :datadir: /etc/puppet/hiera/
:merge_behavior: deeper
:hierarchy:
 - "hosts/%{clientcert}"
 - "domains/%{domain}/environments/%{environment}"
 - "domains/%{domain}"
 - "environments/%{environment}"
 - "default"
```

So, our hiera data are located in `/etc/puppet/hiera` and have following folder structure and lookup order:

- default.yaml - default data, valid and present for all nodes
- environments/{environment}.yaml - data valid for particular puppet environment, in my case desktops and servers
- domains/{dns domain}.yaml - data valid for dns domains, in my case this are physical locations
- domains/{dns domain}/environments/{environment}.yaml - data valid for particular environments inside of dns domains
- hosts/{host fqdn}.yaml - data for particular hosts

Hiera has one limitation with automatically merging hashes, see [this article](/blog/2014/07/05/puppet-hiera-hash-merge-and-automatic-parameter-lookup/) for details how to handle this.

# Modules

It makes sense to have a structure with different module types:

- distribution modules (like modules you install from puppetforge or git)
- some custom modules
- and some site modules

The distribution and custom modules are intended to cover some specific functionality or software, like management of lvm volumes with [puppetlabs-lvm][puppetlabs-lvm] or management of sambe with [thias-samba][thias-samba].
Site modules represent some kind of wrapper modules, which combine the functionality provided by distribution or custom modules. For example we might have a site module ``site_share``, which creates an lvm volume via [puppetlabs-lvm module][puppetlabs-lvm] and then a samba share with [thias-samba][thias-samba].

I manage my dist modules with [librarian][librarian], so I have a Puppetfile and following configuration for librarian

```yaml
# .librarian/puppet/config
---
  LIBRARIAN_PUPPET_DESTRUCTIVE: "false"
  LIBRARIAN_PUPPET_PATH: dist
```

So, in the end we will have following structure for modules:

- Puppetfile
- .librarian/puppet/config
- dist/
  - some distribution/upstream modules
  - ...
- custom/
  - some custom modules
  - ...
- site/
  - some wrapper modules
  - ...

# Environments

Probably we want to have different environments, like staging and production, or probably completely different setups on the different sites.
With newer puppet versions there is a new option [environmentpath][puppet-envpath], which is pretty useful if used together with [r10k][r10k]. So specific environment code is contained within one git branch, which is managed by r10k. It makes also sense to have some kind of a 'global' environment, which can contain some global available modules. The structure within one environment could look like this:

- global
  - environment.conf
  - manifests
  - modules
    - Puppetfile
    - dist/
    - custom/
    - site/
- staging/
  - environment.conf
  - manifests/
  - modules/
    - Puppetfile
    - dist/
    - custom/
    - site/
- production/
  - environment.conf
  - manifests/
  - modules/
    - Puppetfile
    - dist/
    - custom/
    - site/

# Final structure and configuration

In the end, the final structure looks like this

- hiera.yaml
- puppet.conf
- fileserver.conf
- auth.conf
- environments/
  - global
    - environment.conf
    - manifests
    - modules
      - Puppetfile
      - dist/
      - custom/
      - site/
  - staging/
    - environment.conf
    - manifests/
    - modules/
      - Puppetfile
      - dist/
      - custom/
      - site/
  - production/
    - environment.conf
    - manifests/
    - modules/
      - Puppetfile
      - dist/
      - custom/
      - site/
- hiera/
  - default.yaml
  - environments/
    - staging.yaml
    - production.yaml
  - domains/
    - {some domain}.yaml
    - ...
    - environments/
      - staging.yaml
      - production.yaml
  - hosts/
    - {some hosts fqdn}.yaml
    - ...
- secrets/
  - {some hosts fqdn}/
    - some content
    - ...
  - ...

# See too

- [How do you place files on a puppet master?](https://lelutin.ca/article/2012-02-12/how-do-you-place-files-puppet-master)
- [Git workflow and puppet environments](http://puppetlabs.com/blog/git-workflow-and-puppet-environments/)
- [Managing Puppet modules with librarian-puppet](http://blog.csanchez.org/2013/01/24/managing-puppet-modules-with-librarian-puppet/)
- [Librarian][Librarian]
- [R10K][r10k]
- [Rethinking Puppet Deployment](http://somethingsinistral.net/blog/rethinking-puppet-deployment/)
- [Dynamic environments and Hiera](https://groups.google.com/forum/?fromgroups#!topic/puppet-users/ADiJoCCq6_k)
- [Simple puppet module structure](http://www.devco.net/archives/2012/12/13/simple-puppet-module-structure-redux.php)
- [Inheriting defaults from a ::params class in a define type](https://groups.google.com/forum/#!topic/puppet-dev/TKiuoSvd7C0)

[librarian]: http://librarian-puppet.com/
[r10k]: https://github.com/adrienthebo/r10k
[puppet-secret]: https://github.com/TelekomCloud/puppet-secret
[puppetlabs-lvm]: https://github.com/puppetlabs/puppetlabs-lvm
[thias-samba]: https://github.com/thias/puppet-samba
[puppet-envpath]: https://docs.puppetlabs.com/puppet/latest/reference/environments.html
[puppet-hiera]: https://docs.puppetlabs.com/hiera/1/
