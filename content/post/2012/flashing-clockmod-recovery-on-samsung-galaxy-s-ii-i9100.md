+++
date = "2012-04-16T17:15:00+01:00"
tags = [ "android", "galaxysII", "samsung" ]
title = "Flashing clockmod recovery on Samsung Galaxy S II i9100"
+++

You may need custom recovery if you want to do nandroid backups of flash some zip files. I will use Linux to flash it.

<!-- more -->

**Warning: It was working for me. I'm not responsible if you break your device. You are doing this steps at your own risk. The installation of a modified firmware might invalidate the manufacturer's warranty!**


# Get Heimdall Suite

Heimdall Suite is an alternative to Odin for flashing in the download mode.

- Gentoo: add flow to the overlays and emerge heimdall

```bash
layman -a flow
emerge --autounmask-write heimdall
etc-update
emerge heimdall
```

- For other distributions take a look at [offical page of Heimdall Suite](http://www.glassechidna.com.au/products/heimdall/)

# codeworkx's Kernel with the ClockworkMod Recovery

- I've used this kernel with Android 2.3.3, for other Android versions you may use kernels from [this thread](http://forum.xda-developers.com/showthread.php?t=1103399)
- [Download](http://cmw.22aaf3.com/c1/recovery/recovery-clockwork-4.0.1.4-galaxys2.tar) codeworkx's Kernel with the ClockworkMod Recovery 4.0.1.4 and unpack it
- Boot your phone to download mode
  - Switch it off
  - Connect via USB to PC
  - Press and hold Home+Volume Down+Power until you see the "Warning" message
  - Confirm it with Volume Up

- Flash the kernel(to avoid various access problems please do it as root)

```bash
    heimdall flash --kernel zImage
```

- Change to recovery mode
  - Switch your phone off
  - Press and hold Home+Volume Up+Power until you see recovery screen

# See too

- [How to put Samsung Galaxy S II to recovery mode](http://forum.xda-developers.com/wiki/Samsung_Galaxy_S_II_Series#Recovery_Mode)
- [How to flash ClockworkMod Recovery](http://wiki.cyanogenmod.com/wiki/Samsung_Galaxy_S_II:_Full_Update_Guide#Installing_the_ClockworkMod_Recovery)
