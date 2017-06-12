+++
date = "2014-07-06T20:45:13+01:00"
tags = [ "puppet", "ssh", "linux" ]
title = "Easy SSH key management with puppet"
+++

SSH key management is required in each environment.

In this post I want explain how to do it with puppet on the simple way.

I've created a [module][puppet-sshkeys], which is a wrapper around core puppet types User and Ssh_authorized_key. This wrapper enables an easy key management via integration with hiera on puppet. (and it was a good exercise in rspec-puppet:) )

<!-- more -->

# Setup the environment

You will need puppet >=3.6, install it via your package manager, gems or [repositories provided by PuppetLabs][puppetlabs-repos].

Configure hiera like below

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
 - "environments/%{environment}"
 - "default"
```

Create directory structure for hiera

```bash
$ mkdir -p /etc/puppet/hiera/environments
$ mkdir -p /etc/puppet/hiera/hosts
#this symlink is requiered if you want to do some hiera testing on CLI
$ ln -s /etc/puppet/hiera.yaml /etc/hiera.yaml
```

Install the [sshkeys module][puppet-sshkeys]

```bash
$ puppet module install sidorenko-sshkeys
```

# Create the configuration

Place the keys in the default hierarchy, so they will be available for the entire environment(they will not be deployed somewhere, not yet)

```yaml
# /etc/puppet/hiera/default.yaml
---
sshkeys::keys:
  user1@example.com:
    type: 'ssh-rsa'
    key: 'AAAAB3NzaC1yc2EAAAADAQABAAABAQDJzdGydf2tdZYCkBRGx/SnlVKW+9q3Mqtf9vCrs0SaSkwDK4Q36hS40IVgmri2mjKeWFr5p92OgYY1hjZk4LLUAbVV8ItmPLqvmfrkOEwDCzmkbrUVa4BTKePWG0hOGAVYSQkS+1vhsTFhtznJMxsjRVwj8tO3s0fSnaXcovs9d4LwXhRbcDjzrAVRkk2d5/lSbjc/T4ZJ6oMKcGCxq02etJMoSBBQsEfRP/vULqKjoxJ96kb3Y43tU7gRzcVkXAyNqpXie8fD/FopoVi/uHIqkzotkOwztUYNt6C5LwV/W4ds5x3Zl7Jo4kqup2FOCs4oXSC3WxJI5FJ9WuPMtK1r'
  admin1@example.com:
    type: 'ssh-rsa'
    key: 'AAAAB3NzaC1yc2EAAAADAQABAAABAQCqlo0PPGZ2XW1qBFuFgYmsGlT24I+v51tb7cRSAJeBouDPvfqBMBOX84ye4DsW3uRmFNXt/wdAr/QnEAlua5bSagVRC2t9X4lkcrFJSSfEA2J29Lh16pPzOK/HReo8R89wbEKfqrqZG/FNrjMB6YaAxBRJE0O9T6BDsMBCg6b8wb6DRPIKzuEkKkI9ywExVrVFOEANTsdS0oQq8exIlWHmnKwOf1R2Jl1FRgIHnJAfG29EoeY7Q+DlPZOBXqB+xamYj56h6FMb0ZLBOAirXm76bHbqJhzY5RbcW8HrxzvLBY1xfOlP4NMKWIxBNG1j2Je0WPU9gVDnq7/LoS0OuCtR'
  admin2@example.com:
    type: 'ssh-rsa'
    key: 'AAAAB3NzaC1yc2EAAAADAQABAAABAQC9EC7xtYYZZblNgQP/SQ/7NR0fUW+mSMNv6gjDqfhPJ8K4mgqAN4ozvxnHl5k7dfzV4OhB/lIrnjfBg7BIfJjtxcoMNJDSDSmYixX7MzS/Ec35k/ovlxkK5tRKdhZHKYigLSUS2gE30l0804FeCj36O19UBeArrSXghsaKELFuE2EqUGz9kZ9WZW9SVDdJKuTSuij9GspIRCdhMX/s6GQOiycremqtnHf1xuZ22bSkbuAAYvPxQTvsxCMtykE4iqdJ8xhWeO+CZZMWn11AEv1FscwbirbjkjXz02D57BaeOwlU13oZIfA6Ko4SkMa9FuhNrtn4ctWb5jBep9xzyZUR'
```

Now assign the both keys from admins to the user admin in the environment serverfarm

```yaml
# /etc/hiera/environments/serverfarm.yaml
---
sshkeys::users:
  admin:
    home: /home/admin
    keys:
      - admin1@example.com
      - admin2@example.com
```

Now you want to assign the user1 key to the user1 on the system cool-system.example.com.

```yaml
# /etc/hiera/hosts/cool-system.example.com.yaml
sshkeys::users:
  user:
    home: /home/user
    keys:
      - user1@example.com
```

If you remove some key assignments, but the User stays managed, they keys will get automatically removed from the system.

More detailed information and examples can be found in the module [README][puppet-sshkeys-readme] file.

[puppet-sshkeys]: https://github.com/artem-sidorenko/puppet-sshkeys/
[puppet-sshkeys-readme]: https://github.com/artem-sidorenko/puppet-sshkeys/blob/master/README.md
[puppetlabs-repos]: http://docs.puppetlabs.com/guides/puppetlabs_package_repositories.html
