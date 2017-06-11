+++
date = "2012-12-30T18:02:04+01:00"
tags = [ "android", "galaxysII", "samsung" ]
title = "Flashing Android 4 on Samsung Galaxy S II with heimdall"
+++

We are using only Linux and no Odin and VirtualBox to update Galaxy S II to Android 4 alias Ice Cream Sandwich.

<!-- more -->

**Heimdall 1.3.2 seems to be broken, so use heimdal < 1.3.2. [More information](https://github.com/Benjamin-Dobell/Heimdall/issues/32)**

# Install Heimdall

Heimdall Suite is an alternative to Odin for flashing in the download mode.

- Gentoo: add flow to the overlays and emerge heimdall

```bash
layman -a flow
emerge --autounmask-write heimdall
etc-update
emerge heimdall
```

- For other distributions take a look at [offical page of Heimdall Suite](http://www.glassechidna.com.au/products/heimdall/).

# Get ICS and some other images

- Download the firmware, [from Hotfile](http://hotfile.com/dl/149514867/aea2f22/I9100XXLPQ_I9100OXALPQ_XEO.zip.html)
- If you are going to root, download [CF-Root](http://download.chainfire.eu/152/CF-Root/SGS2/CF-Root-SGS2_XX_XEO_LPQ-PROPER-v5.4-CWM5.zip)
- [or complete androidnext.de package via torrent](http://dl.androidnext.de/samsung.galaxy.s.ii.ics.package.-.xxlpq.firmware.root.odin.-.androidnext.de.zip.torrent)

# Flash it

Unpack the file "I9100XXLPQ_I9100OXALPQ_I9100XXLPQ_HOME.tar.md5"

Boot your phone to download mode

- Switch it off
- Connect via USB to PC
- Press and hold Home+Volume Down+Power until you see the “Warning” message
- Confirm it with Volume Up

Flash it with heimdall (do it with as root to avoid access problems to usb devices)

```bash
heimdall flash --primary-boot boot.bin --cache cache.img \
--factoryfs factoryfs.img --hidden hidden.img --modem modem.bin \
--param param.lfs --secondary-boot Sbl.bin --kernel zImage
```

# See too

- German [Samsung Galaxy S2: Ice Cream Sandwich von Hand installieren](http://www.androidnext.de/howto/samsung-galaxy-s2-ice-cream-sandwich-howto/)
- [Flashing with Heimdall](http://www.darkyrom.com/community/index.php?threads/flashing-with-heimdall.2414/)
- [Flashing stock firmware with Heimdall](http://www.poempelfox.de/blog/2011/10/)
