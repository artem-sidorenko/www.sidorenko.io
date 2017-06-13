+++
date = "2016-04-12T15:20:52+01:00"
tags = [ "android" ]
title = "Flashing BQ Aquaris X5"
+++

Since couple of months the Aquaris X5 from BQ is [available in Germany](http://www.amazon.de/C000079-Aquaris-Smartphone-Megapixel-schwarz/dp/B0173I64EW/).
Its a pretty cool device, besides the fair price it has Dual-SIM and MicroSD support and a battery run-time up to several days.
Another interesting thing is this smartphone is offered in two editions: with Android or with CyanogenOS software flashed.
[CyanogenOS](https://cyngn.com/cyanogen-os) is the commercial version of [CynanogenMod](http://cyanogenmod.org/) and its quite promising especially from the view on software updates and security patches with a stock software.

This post will describe the (re-)flash procedure of Aquaris X5 with stock Android, CyanogenOS or CyanogenMod.
So still if you want to stay on the stock software, but want to switch from Android to CyanogenOS or back, this post might be useful for you.

This guide covers the steps required on Linux, but it should work on the similar way on Windows. Hoewever I can't verify it as I do not have any Windows system.

<!--more-->

# Allow OEM unlocking

Per default is not possible to flash something via fastboot and you have to allow OEM unlocking first. This procedure apply to Android, CyanogenOS and CyanogenMod on the same way:

- Boot your device
- Go to Settings -> About phone -> Tap Build number several times to enable development settings
- Go to Settings -> Developer Options -> Enable OEM unlocking
- Switch off your device

# Install the requiered software

Install the fastboot via packages:

```bash
# Example for Ubuntu
$ apt-get install android-tools-fastboot
```

(If you are on Windows, you probably need to download the drivers from [BQ Aquaris X5 support page](http://www.bq.com/uk/support/aquaris-x5) and install them)

# Boot to fastboot and unlock the device

The magic key combination is different dependending on the current OS flashed on the device.
Switch off your device then press and hold following buttons until you see the hint about fastboot mode

- Android : volume down + power
- CyanogenOS / CyanogenMod: volume up + power

Unlock the bootloader (**attention, this will wipe all your data!**):

```bash
$ fastboot oem unlock-go
```

Your device gets restarted and you will see the wipe process.

# Flash the OS

## Flashing Android

Via the following steps you can restore the phone to the stock Android firmware.

- Download the stock Android firmware on the [BQ Aquaris X5 support page](http://www.bq.com/uk/support/aquaris-x5)
- Switch to the fastboot mode like desribed [above](#boot-to-fastboot-and-unlock-the-device)
- Unzip the firmware, change to the folder with unzipped files and flash them

```bash
# This steps are valid for Firmware 3.2.9
# You can check the file *_fastboot_all_images.bat for latest steps
$ fastboot flash aboot emmc_appsboot.mbn
$ fastboot flash abootbak emmc_appsboot.mbn
$ fastboot erase DDR
$ fastboot flash sbl1 sbl1.mbn
$ fastboot flash sbl1bak sbl1.mbn
$ fastboot flash tz tz.mbn
$ fastboot flash tzbak tz.mbn
$ fastboot flash hyp hyp.mbn
$ fastboot flash hypbak hyp.mbn
$ fastboot flash rpm rpm.mbn
$ fastboot flash rpmbak rpm.mbn
$ fastboot flash modem NON-HLOS.bin
$ fastboot flash cache cache.img
$ fastboot erase splash
$ fastboot flash splash splash.img
$ fastboot flash system system.img
$ fastboot flash recovery recovery.img
$ fastboot flash userdata userdata.img
$ fastboot flash boot boot.img
# optionally you can lock the device
$ fastboot oem lock
# reboot
$ fastboot reboot
```

## Flashing CynanogenOS

Via following steps you can restore the phone to the stock CyanogenOS firmware.

- Download the stock CyanogenOS firmware on the [Cyanogen support page](https://cyngn.com/support)
- Switch to the fastboot mode like desribed [above](#boot-to-fastboot-and-unlock-the-device)
- Unzip the firmware, change to the folder with unzipped files and flash them

```bash
# This steps are valid for Firmware cm-12.1-YOG4PAS5UI-paella-signed-fastboot-3b209bb276
# You can check the file falsh-all.bat for latest steps
$ fastboot flash boot boot.img
$ fastboot flash system system.img
$ fastboot flash recovery recovery.img
$ fastboot flash splash splash.img
$ fastboot flash aboot emmc_appsboot.mbn
$ fastboot flash modem NON-HLOS.bin
$ fastboot flash userdata userdata.img
# optionally you can lock the device
$ fastboot oem lock
# reboot
$ fastboot reboot
```

## Flashing CynanogenMod

If you want to run CyanogenMod, you will have to install the recovery first:

- [Download](https://dl.twrp.me/piccolometal/twrp-3.0.0-0-piccolometal.img) TeamWin Recovery
- Switch to the fastboot mode like desribed [above](#boot-to-fastboot-and-unlock-the-device)
- Flash the recovery via fastboot

```bash
$ fastboot flash recovery twrp-3.0.0-0-piccolometal.img
```

Now you can proceed to the CyanogenMod installation via recovery:

- [Download](https://download.cyanogenmod.org/?device=paella) CyanogenMod for Aquaris X5
- Place the CyanogenMod package to some sdcard via PC
- Switch off your device and boot to recovery, key combinatations depend on the currently installed OS:
  - Android : volume up + power
  - CyanogenOS / CyanogenMod: volume down + power
- Probably you might need to wipe the data partition: Wipe -> Format Data
- Flash the zip via Team Win Recovery: Install -> select the zip file and choose `install image`
- Reboot the device

# See too

- [CynogenOS flashen (German)](http://www.android-hilfe.de/thema/bq-aquaris-x5-androidversion-cyanogen-os-flashen.742188/)
- [How to Install CyanogenMod on the BQ Aquaris X5](https://wiki.cyanogenmod.org/w/Install_CM_for_paella)
