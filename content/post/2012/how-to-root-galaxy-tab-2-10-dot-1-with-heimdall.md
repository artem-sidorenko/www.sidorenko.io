+++
date = "2012-07-11T21:19:05+01:00"
tags = [ "android", "samsung" ]
title = "How to root Galaxy Tab 2 10.1 with heimdall"
+++

We are going to install ClockworkMod Recovery on the Galaxy Tab 2 10.1 (GT-P5100) with heimdall. After that you can root your stock image or install CyanogenMod on your tab. If you have another version of Galaxy Tab 2 you will need other images for it, for links take a look at the links at the bottom.

<!--more-->

**Warning: It was working for me. I'm not responsible if you break your device. You are doing this steps at your own risk. The installation of a modified firmware invalidates the manufacturer's warranty! You have been warned!**

# Flashing recovery

What we need:

- [Heimdall v.1.3.1](http://www.glassechidna.com.au/products/heimdall/)
- [Recovery image (Odin package)](http://teamhacksung.org/download/samsung/GT-P5100/GT-P5100_ClockworkMod-Recovery_5.5.0.4.tar)

Lets go

- Install heimdall, for gentoo installation instructions take a look to the instructions [here](/blog/2012/12/30/flashing-android-4-on-samsung-galaxy-s-ii-with-heimdall/)
- Unpack the tar with recovery image
- Disconnect your tab from PC and shut it down
- Press Power+Volume Up
- Confirm the warning dialog with Volume Down
- Connect the tab to PC
- Flash the recovery (please start heimdall with root rights to avoid any access problems)

```bash
heimdall flash --recovery recovery.img
```

- Disconnect your tab
- You can boot to recovery if our press Power+Volume Down, release power button when you see Galaxy Tab logo and hold Volume Down for the next 5 seconds. Wait for recovery, it takes some time.
- You can now proceed with rooting of stock image or with installing of CyanogenMod

# Rooting stock image

What we need:

- [Root installation zip](http://forum.xda-developers.com/attachment.php?attachmentid=1113246&d=1339079326)

Lets go

- Upload the CWM package to sdcard (yes, to the external microSD)
- Boot to recovery and select "install the zip"
- Select CWM package, that's all

# Flashing CyanogenMod with Google Apps

What we need:

- [CyanogenMod for GT-P5100](http://builds.teamhacksung.org/?dir=&download=cm-9-20120706-EXPERIMENTAL-p5100-CODEWORKX.zip)
- [Google Apps for CM9](http://goo.im/gapps/gapps-ics-20120429-signed.zip)

Lets go

- Upload the both zip files to the external sdcard
- Boot to recovery and install the zips
- Do a "wipe data / factory reset"
- Reboot

# See too

- [ClockworkMod Recovery 5.5.0.4 for the Samsung Galaxy Tab 2 (GT-P51XX)](http://forum.xda-developers.com/showthread.php?t=1722063)
- [CyanogenMod 9 4.0.4 for GT-P51xx](http://forum.xda-developers.com/showthread.php?t=1722099)
- [Google Apps for CyanogenMod](http://goo.im/gapps)
- [ICS 4.0.3 Stock Safe / Recovery / Root (P5100XWALD7)](http://forum.xda-developers.com/showthread.php?t=1686514)
