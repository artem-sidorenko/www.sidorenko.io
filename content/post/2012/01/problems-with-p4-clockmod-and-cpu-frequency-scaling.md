+++
date = "2012-01-29T12:00:00+01:00"
tags = [ "linux", "kernel", "hardware" ]
title = "Problems with P4 Clockmod and CPU frequency scaling"
+++

If you are going to use CPU frequency scaling on P4 you will probably use p4-clockmod and get in trouble with ondemand governor like me.
If you try to switch to the ondemand governor you can see following message in dmesg:

```text
ondemand governor failed, too long transition latency of HW, fallback to performance governor
```

<!--more-->

# Solution

Just change the following string in kernel sources `arch/x86/kernel/cpu/cpufreq/p4-clockmod.c` like here

**old string**

```c
policy->cpuinfo.transition_latency = 1000000;
```

**new string**

```c
policy->cpuinfo.transition_latency = 10000001;
```

# See also

- [Gentoo Bug 287463](https://bugs.gentoo.org/show_bug.cgi?id=287463)
