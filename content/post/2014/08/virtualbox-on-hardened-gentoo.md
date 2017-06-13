+++
date = "2014-08-27T18:11:38+01:00"
tags = [ "security", "gentoo", "virtualbox" ]
title = "Virtualbox on hardened gentoo"
+++

I wanted to setup [phpVirtualBox][phpvirtualbox] on my new [Intel NUC][intel_nuc], which is running gentoo-hardened. Unfortunately VirtualBox can't run with couple of grsecurity/pax flags enabled in kernel. To get VirtualBox running you have to disable following kernel config flags:

- CONFIG_PAX_KERNEXEC
- CONFIG_PAX_RANDKSTACK
- CONFIG_PAX_MEMORY_UDEREF
- CONFIG_GRKERNSEC_HIDESYM

and to enable:

- CONFIG_PAX_ELFRELOCS (if you have CONFIG_PAX_MPROTECT)

[intel_nuc]: http://www.intel.com/content/www/us/en/nuc/overview.html
[phpvirtualbox]: http://sourceforge.net/projects/phpvirtualbox/
