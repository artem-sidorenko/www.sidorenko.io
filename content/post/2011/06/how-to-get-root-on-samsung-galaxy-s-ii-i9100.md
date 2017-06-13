+++
date = "2011-06-28T20:48:09+01:00"
tags = [ "android", "samsung" ]
title = "How to get root on Samsung Galaxy S II I9100"
+++

Why root? I want to use some advanced features like DroidWall(frontend for iptables), Titanium Backup, OpenVPN, so I need root.

<!--more-->

**Warning: It was working for me. I'm not responsible if you break your device. You are doing this steps at your own risk. The installation of a modified firmware may invalidate the manufacturer's warranty! You have been warned!**

# What do we need?

I used a linux PC and virtual Windows in a VirtualBox VM.
Please check that you are running [Android 2.3.3](http://androidadvices.com/how-to-update-samsung-galaxy-s-ii-i9100-to-gingerbread-xwkdd-2-3-3-firmware/).

- Download [insecure kernel](http://androidadvices.com/wp-content/uploads/2011/05/XWKDD_insecure.tar)
- Download [Odin](http://androidadvices.com/wp-content/uploads/2011/05/Odin3-v1.85.zip)
- Download [Su update package](http://downloads.androidsu.com/superuser/su-2.3.2-efgh-bin-signed.zip)

# Flashing the insecure kernel

- Switch off your phone
- Press and hold Volume Down+Ok and press the power button. You should see the download picture
- Start Odin in the VM, select the in the PDA section the **XWKDD_insecure.tar**
- Connect your mobile via USB tunneling and choose start after you see the com port in the odin frontend.
- Wait for reboot...

# Getting root

- Install "Superuser" from Android Market
- Unpack "system/bin/su" from su-2.3.2-efgh-bin-signed.zip
- Push su to the mobile via adb:

```bash
/opt/android-sdk-update-manager/platform-tools/adb push su /sdcard/su
```

- Start adb shell and remount the /system read write

```bash
/opt/android-sdk-update-manager/platform-tools/adb shell
mount -o remount,rw /dev/block/mmcblk0p9 /system
```

- Copy su

```bash
cat /sdcard/su > /system/bin/su
chmod 06755 /system/bin/su
```

- Remount /system read only and remove su from sdcard

```bash
mount -o remount,ro /dev/block/mmcblk0p9 /system
rm /sdcard/su
exit
```

That's all;-)

# See also

- http://androidadvices.com/how-to-root-samsung-galaxy-s-2-i9100-tutorial/
- http://www.addictivetips.com/mobile/how-to-root-samsung-galaxy-s-2-i9100/
- http://androidsu.com/superuser/
