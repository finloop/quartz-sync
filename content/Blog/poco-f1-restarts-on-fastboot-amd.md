---
title: "Poco F1 Restarts on Fastboot Amd"
date: 2023-07-08T21:15:20+02:00
publish: true
---

## Poco F1 - Restarts on Fastboot (AMD)

I tried to flash custom recoery for my Poco F1 (codename "beryllium"). I installed ADB drivers and drivers for my phone. Windows device manager said everything was ok so I procedded and entered fastboot mode (Volume down + Power). I entered command:

```sh
fastboot devices
```

And the fastboot logo disappered and was replaced with text in top left corner: "Press any key to shutdown". Then I remembered that the guide said this:

> Also, have a Xiaomi Account with Mobile Number of Sim1 ready for sign-in whenever required prior to Bootloader Unlocking and make sure you have a couple of Mobile Data MBs at least. And an INTEL PC/Laptop. For AMD you may need to do via WSL Ubuntu. If you encounter issues try to power off via the options available and boot straight to the fastboot or recovery depending in whichever stage you are.

Source: [[ROM][13][OFFICIAL] PixelOS [AOSP][STABLE][16/02/2023]](https://forum.xda-developers.com/t/rom-13-official-pixelos-aosp-stable-16-02-2023.4401621/)

I have AMD CPU so I had to do something else. I found a solution on XDA Developers forum.

## Solution

Create a new \*.bat file and execute it. It should contain:

```sh
@echo off
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\usbflags\18D1D00D0100" /v "osvc" /t REG_BINARY /d "0000" /f
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\usbflags\18D1D00D0100" /v "SkipContainerIdQuery" /t REG_BINARY /d "01000000" /f
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\usbflags\18D1D00D0100" /v "SkipBOSDescriptorQuery" /t REG_BINARY /d "01000000" /f

pause
```

This will add some registry keys that will allow you to use fastboot on your phone on AMD CPU.

Credit: [[FIX] Fastboot not working on Ryzen CPU](https://forum.xda-developers.com/t/fix-fastboot-not-working-on-ryzen-cpu.4182879/)
