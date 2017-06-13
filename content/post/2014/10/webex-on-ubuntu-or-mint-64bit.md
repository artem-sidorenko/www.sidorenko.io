+++
date = "2014-10-28T17:20:22+01:00"
tags = [ "linux", "mint", "ubuntu" ]
title = "WebEx on Ubuntu or Mint 64bit"
+++

If you want to use WebEx on Ubuntu/Mint 64 bit, you will see: it works.
But you won't be able to do screen sharing or probably even to see webcam video streams.

The only one solution is to use 32bit oracle java incl. browser plugin, so browser has to be 32bit as well.
If you want to keep 64bit browser on the system and to use 32bit setup only for WebEx, then this howto is for you:)

<!--more-->

- We will create a folder ``/opt/webex-firefox`` and place 32bit firefox and java there.
- Firefox will still use the default folders like ``~/.mozilla`` on your system, so we will create a separate profile
- First of all download the latest 32bit
  - [Mozilla Firefox](https://www.mozilla.org/en-US/)
  - [Oracle Java](http://www.java.com/en/download/linux_manual.jsp?locale=en)
- Unpack and create the structure (as root)

```bash
#install 32 libraries with dependencies
$ apt-get install libgtk2.0-0:i386 libpangoxft-1.0-0:i386 libpangox-1.0-0:i386 libxmu6:i386 libxtst6:i386 libdbus-glib-1-2:i386

$ mkdir /opt/webex-firefox
$ cd /opt/webex-firefox
#unpack it
$ tar xfj firefox-33.0.1.tar.bz2
$ tar xfz jre-8u25-linux-i586.tar.gz
$ rm firefox*.tar.bz2
$ rm jre*.tar.gz
$ mv jre1.8.0_25 java

#create symlink for java plugin
$ cd firefox/browser
$ mkdir plugins
$ cd plugins
$ ln -s ../../../java/lib/i386/libnpjp2.so .
$ cd ../../../

#create a starter script, as we need java command from our setup
$ echo "#!/bin/sh" > startfirefox
$ echo "export PATH=\"$(pwd)/java/bin:$PATH\"" >> startfirefox
$ echo "$(pwd)/firefox/firefox -P webex -new-instance" >> startfirefox
$ chmod +x startfirefox
```

- Start firefox via ``/opt/webex-firefox/startfirefox`` and create ``webex`` profile
- Thats all:)

See too:

- [Running 32-bit Firefox with sun-jre in 64-bit Ubuntu](http://askubuntu.com/questions/111947/running-32-bit-firefox-with-sun-jre-in-64-bit-ubuntu)
- [WebEx System Requirements](https://support.webex.com/webex/v1.1/support/en_US/rn/system_rn.htm)
