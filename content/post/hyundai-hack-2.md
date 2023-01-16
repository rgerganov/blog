+++
title = "Hyundai Head Unit Hacking"
date = "2023-01-17T11:55:12+02:00"
tags = []
topics = []
description = "Standard-class Gen5 navigation"
+++

In the [previous post](https://xakcop.com/post/hyundai-hack/) I have shown how to crack the official firmware for Hyundai Tucson 2020
and reverse engineer it. At the end I was thinking that I can simply modify the update package,
zip it again with the same password and push it to the car. But it turned out it is not that simple.
The update package is signed with an RSA key which corresponds to the `daudio.x509.pem` certificate
and this signature is checked during the update. This is part of the Android OTA update process
which is being used for updating the firmware of the entire unit (not just the car navigation).
Unlike the RSA key for [Ioniq 2021](https://programmingwithstyle.com/posts/howihackedmycar/), 
this key cannot be found online (at least I haven't found it).
How can we get access to the head unit in this case? I was thinking either of these two options:

 * find an exploitable bug in one of the applications
 * find an exploitable bug in the Linux kernel; the head unit is running Linux 3.1.10, so this looked feasible

I had no luck with both of them. Fortunately, I found some new information which allowed me to root the head unit.

## New findings
First and foremost, I realized that Hyundai is shipping the same firmware to a variety of cars.
My car had the so called "Standard-class Gen5 navigation" which looks like this:

[{{% fluid_img class="pure-u-1-1" src="/images/gen5.png" %}}](/images/gen5.png)

They call it "navigation" but it is basically the firmware of the entire head unit.
The same firmware is shipped on different Hyundai, KIA and Genesis models manufactured in the 2018-2021 time frame.

The head unit is running on [Telechips TCC893X SoC](https://www.cnx-software.com/2014/01/20/telechips-tcc893x-dual-core-arm-cortex-a9-cortex-m3-socs-tcc8930-tcc8933-tcc8935/)
and its SDK has been [leaked](https://www.cnx-software.com/2014/02/12/telechips-tcc892xtcc893x-android-4-2-2-sdk-leaked/) on the internet.
There is a secret recovery mechanism which is triggered by holding the POWER button (left knob) and the MAP button upon start:

[{{% fluid_img class="pure-u-1-1" src="/images/head_unit.jpg" %}}](/images/head_unit.jpg)

I tried it on my Hyundai Tucson 2020 and I got this nice error on the car screen:

[{{% fluid_img class="pure-u-1-1" src="/images/uboot.jpg" %}}](/images/uboot.jpg)

Apparently the recovery mechanism is looking for some encrypted files on the USB drive.
A simple grep for these strings leads to the `lk.rom` file from the update package which I have been ignoring until now.
Let's load it in Ghidra and see what's going on.

## Reversing `lk.rom`
LK stands for "little kernel", a small [open-source kernel](https://github.com/littlekernel/lk) which is used in many embedded platforms.
The head unit is loading `lk.rom` at address `0x82000000`. After setting the correct start address in Ghidra, we can
easily identify `printf` functions which print a lot of useful debug info.
Tracing the message `"[DEBUG] U-Boot Recovery Button pushed .... \n"` leads to:

[{{% fluid_img class="pure-u-1-1" src="/images/lk-recovery.png" %}}](/images/lk-recovery.png)

Looks like the recovery mechanism is part of u-boot and its entry point is the function at `0x820589a8`:

[{{% fluid_img class="pure-u-1-1" src="/images/uboot_start.png" %}}](/images/uboot_start.png)

Using the debug message at line 14, we can easily infer that this function copies the u-boot code to `0x80000000` and starts it.
`PTR_DAT_82058a38` is the beginning address of u-boot and `PTR_DAT_82058a3c` is the end address:

[{{% fluid_img class="pure-u-1-1" src="/images/uboot_ptrs.png" %}}](/images/uboot_ptrs.png)

Using these addresses, we can "extract" the u-boot code from `lk.rom` with the following command:

```
$ dd if=lk.rom skip=$((0x1055c)) count=$((0x57894-0x1055c)) bs=1 of=uboot.rom
```

And then analyze `uboot.rom` as a separate binary with start address `0x80000000` in Ghidra.

## Reversing `uboot.rom` (part of `lk.rom`)
There are again many debug strings which help a lot to understand what's going on.
The recovery mechanism is looking for the following files on the USB drive:
 * `security_force/encrypt_lk.rom`
 * `security_force/encrypt_boot.img`
 * `security_force/encrypt_system.img`
 * `security_force/encrypt_recovery.img`
 * `security_force/encrypt_splash.img`
 * `security_force/encrypt_partition.dat`

There is also `security_force/file_info` which contains the name, size and CRC32 checksum for each of the above files.
These files (with the exception of `encrypt_partition.dat`) correspond to the files we have found in the update package:

[{{% fluid_img class="pure-u-1-1" src="/images/update.zip.png" %}}](/images/update.zip.png)

They must be encrypted with AES-128-CBC using key=")1Zorxo^fAhGlh$#" and IV="aoAkfwk+#1.6G{dE".
Only `system.ext4` must be converted to sparse image before the encryption.

## Patching system.ext4
Assuming that we can flash whatever we want with the recovery mechanism, what would be the minimal patch for the official firmware
which will give us some kind of access to the head unit? While looking for exploitable bugs in the stock applications, I found
a hidden menu in the Engineering Mode which enables ADB:

[{{% fluid_img class="pure-u-1-1" src="/images/engmode_adb.png" %}}](/images/engmode_adb.png)

The boolean flag `mDispAdb` can be switched by tapping 5 times in the bottom right corner of the 3rd page of "Module Info".
However, if `ADB_HIDE_FEATURE` is present this flag is ignored and visibility is always set to 8 which means [GONE](https://developer.android.com/reference/android/view/View#GONE).
The `ADB_HIDE` feature is set by default as we can see in `system.ext4`:
```xml
$ cat /tmp/car/etc/permissions/com.hkmc.software.engineermode.adb_hide.xml 
<permissions>
    <feature name="com.hkmc.software.engineermode.adb_hide" />
</permissions>
```
Well, let's delete this feature, create a recovery package and push it to the car. Long story short, that worked!
With this simple change we have successfully enabled ADB on Kia Stinger 2020 and connected to it over USB!

[{{% fluid_img class="pure-u-1-1" src="/images/adb_menu.jpg" %}}](/images/adb_menu.jpg)

## Getting root shell
Now when we have an ADB shell how to become root? Turns out there is a convenient setuid binary called "amossu" in the stock firmware:
```
$ ls -la bin/amossu
-rwsr-sr-x 1 root root 37216 Oct  6 08:29 bin/amossu
```
It simply does:
```c
setgid(0);
setuid(0);
execv("/system/bin/sh",__argv);
```

## Tooling
I have released a small tool and instructions how to create custom firmware for cars with Gen5 navigation.
You can find it [here](https://github.com/rgerganov/gen5fw).
So far we have successfully verified the entire process on Kia Stringer 2020 (thanks to Ali Al-Rawi).

## Final thoughts
I hope this hack will allow creating some interesting mods for Gen5 cars.
For example, I'd love to see an app which records a video stream from the car's camera and saves it on a USB stick.
Of course, the ultimate goal remains running Doom on the head screen :)

If you have any comments or feedback, you can leave them on [Github](https://github.com/rgerganov/gen5fw/discussions/1).

