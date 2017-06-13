+++
date = "2011-04-27T21:02:33+01:00"
tags = [ "android", "htc" ]
title = "How to root HTC Desire"
+++

Why root? I want to use some advanced features like DroidWall(frontend for iptables), Titanium Backup, OpenVPN, so I need root.

<!--more-->

**Warning: It was working for me. I'm not responsible if you break your device. You are doing this steps at your own risk. The installation of a modified firmware invalidates the manufacturer's warranty! You have been warned!**

We are going to get root on HTC Desire GSM and to install following things:

- ROM - VillainRom 1.0
- S-OFF - Alpharev
- Recovery - Amon RA 2.0.0

We are using following software on PC:

- OS Linux
- VirtualBox

# Get root

[Download](http://downloads.unrevoked.com/recovery/3.32/reflash.tar.gz) [unrevoked3 recovery flash tool](http://unrevoked.com/recovery/). Unpack and start it

```bash
tar xfz reflash.tar.gz
chmod +x reflash
./reflash
```

Switch on the "usb debugging" on your device: "Settings->Applications->Development->USB debugging".
Connect your mobile to PC via USB and follow the instructions of unrevoked3 recovery.

You should get root and clockworksmod recovery image installed on your mobile.

# S-OFF

HTC has implemented security flag, so every image (recovery, rom) will be checked for HTC signature.
We want to flash Amon RA Recovery, so we need to switch it off (S-OFF).

[Download](http://alpharev.nl/alpharev.iso) this boot iso with Alpharev. Burn it and boot your PC from this image or use VirtualBox with USB tunneling. Follow the instructions and you will get S-OFF on your device.

If everything was Ok you've got a new alpharev splash image. To flash the original HTC splash you need fastboot([download](http://developer.htc.com/adp.html)) and original splash([download](http://alpharev.nl/desire_stock_splash1.img)). Save both in the same folder. Power off your mobile, connect it to PC and boot it to fastboot "power+back". Flash the splash:

```bash
unzip fastboot.zip
chmod +x fastboot
./fastboot flash splash1 desire_stock_splash1.img
```

# Amon-RA Recovery

Why not the ClockworkdMod recovery? Amon-RA has more features.
[Download](http://files.androidspin.com/downloads.php?dir=amon_ra/RECOVERY/&file=recovery-RA-desire-v2.0.0.img) the [Amon-RA Desire Recovery 2.0.0](http://forum.xda-developers.com/showthread.php?t=839621) and save it in the same fodler with fastboot. Power off your mobile, connect it to PC and boot it to fastboot "power+back". Flash the recovery:

```bash
./fastboot flash recovery recovery-RA-desire-v2.0.0.img
```

# VillainROM

[Download](http://www.villainrom.co.uk/releases/desire/VillainROM1/VillainROM1-signed.zip) [VillainRom for Desire](http://www.villainrom.co.uk/forum/showthread.php?2917-ROM-28-09-10-VillainROM-1.0-for-the-Desire-%28Sense-UI-2.2-Froyo%29&s=e861d2e973db4648a1782e9af25d85c0) save it on the SD card, boot into recovery and flash it.

# See too

- [Unrevoked3 recovery documentation](http://unrevoked.com/rootwiki/doku.php/public/unrevoked3)
- [S-OFF with alpharev](http://alpharev.nl/)
- [Fastboot documentation](http://wiki.cyanogenmod.com/index.php?title=Fastboot)
