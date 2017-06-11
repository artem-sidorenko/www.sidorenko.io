+++
date = "2011-04-25T22:01:00+01:00"
tags = [ "freebsd" ]
title = "How to batch delete old files after installworld"
+++

After you have build and installed the world, you want probably to clean up your system.
If you do it on the way described in the Makefile, and start **make delete-old** you will be prompted for configuration for each file have to be deleted.

<!--more-->

To do a batch delete without confirmations simple set **BATCH_DELETE_OLD_FILES**:

```bash
make -DBATCH_DELETE_OLD_FILES delete-old
make -DBATCH_DELETE_OLD_FILES delete-old-libs
```

See also:

- [make delete-old question](http://lists.freebsd.org/pipermail/freebsd-questions/2007-November/161890.html)
