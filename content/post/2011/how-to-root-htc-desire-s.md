+++
date = "2011-07-14T20:31:10+01:00"
tags = [ "android", "htc" ]
title = "How to root HTC Desire S"
+++

Why root? I want to use some advanced features like DroidWall(frontend for iptables), Titanium Backup, OpenVPN, so I need root. With root you can also remove some annoying apps installed by vendor.

I was using Linux for it, but it should work at the same way with Windows.

<!-- more -->

**Warning: It was working for me. I'm not responsible if you break your device. You are doing this steps at your own risk. The installation of a modified firmware invalidates the manufacturer's warranty! You have been warned!**

- Boot to Hboot (hold Volume Down and press Power), check for Hboot version. AlphaRev X is working only with HBoot.0.98.0000.
- Flash HBoot and S-OFF
  - Start you mobile and check the settings for USB debugging(Settings-Applications-Development-USB Debugging)
  - Fast boot setting should be off(Settings-Power-Fast boot)
  - [Download](http://alpharev.nl/x/beta/) AplhaRev X and start it
  - After you select download you should get a popup, if not check your Noscript settings.
  - Connect you mobile via USB
  - Now you should get the serial from the application, type this serial on the alpharev website and you get the beta key
  - Say no for installation of recovery
  - Your Desire S is now S-OFF

- Flashing recovery
  - [Download](http://bitly.com/iDADwL?r=bb) Clockworkmod Recovery v3.2.0.0
  - Rename it to PG88IMG.zip and place on the sdcard
  - Boot to the Hboot(Volume Down+Power)
  - Confirm the recovery update
  - Reboot and remove PG88IMG.zip from sdcard

- Installing Superuser und su binary
  - [Download](http://androidsu.com/superuser/) su package(not just the binary! filename should be like %%su-x.y.z-efgh-signed.zip%%) and save it on the sdcard
  - Reboot to the recovery(boot in hboot and select recovery)
  - Choose "install zip from sdcard"-"choose zip from sdcard"-"su-x.y.z-efgh-signed.zip"
  - Reboot
  - Install Superuser from Market

That's all!

# See also

- [AlphaRevX Beta](http://alpharev.nl/x/beta/)
- [Anleitung Bootloader entsperren (S-OFF), CustomROM installieren und Root Zugriff](http://www.android-hilfe.de/root-hacking-modding-fuer-htc-desire-s/122244-anleitung-bootloader-entsperren-s-off-customrom-installieren-root-zugriff.html)
- http://lbc-mod-android-dev.de/custom-recovery/
- http://androidsu.com/superuser/
