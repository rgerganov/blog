+++
title = "Hacking Hyundai Tucson 2020"
date = "2022-08-12T10:00:13+03:00"
tags = []
topics = []
description = ""
+++

I bought Hyundai Tucson 2020 two years ago and recently I found [great series](https://programmingwithstyle.com/tags/hyundai/) of blog posts on how to hack Hyundai Ioniq 2021 by greenluigi1. Unfortunately, the methods described there didn't work for me. My car is running the previous generation of D-Audio which is quite different from D-Audio 2V described by greenluigi1. For reference, these are the exact versions of the firmware/software which I have:

[{{% fluid_img class="pure-u-1-1" src="/images/hyundai-ver.jpg" %}}](/images/hyundai-ver.jpg)

I also found the password protected Engineering Mode which appears by tapping 5 times on the left from the Update button and 2 times on the right:

[{{% fluid_img class="pure-u-1-1" src="/images/engmode-pwd.jpg" %}}](/images/engmode-pwd.jpg)

However, none of the publicly available passwords worked (1200, 2400, 3802, current time, etc.).

So I decided to download the firmware and try to reverse it by myself.

Firmware updates
---
The firmware is available from [update.hyundai.com](https://https://update.hyundai.com/). You have to download and install Navigation Updater and then select your car model. The updater downloads all the files and prepares an SD card or USB drive. For my Hyundai it downloaded 23GB of data which includes both navigation updates and firmware. In the root folder of the USB drive there is `2018_20_Tucson_EU.ver` file which lists all of the files in the update:
```text
+|22Q1|TLFL.EUR.SOP.V126.220421.STD_M|HM|2018_20_Tucson_EU|1477|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur|vr.img|10|623073973|97644544|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update|checksum|10|-755847668|1888|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update|info.ini|10|-2103668610|256|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update|update.ini|10|-1246074861|1883|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\gps|gps.inf|10|-1353122717|69|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\gps|gps_module.bin|10|-1514928853|561740|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\micom|micom.inf|10|1086259435|68|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\micom|micom_sw.bin|10|-468017694|1048452|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|qb_data.sparse.img|10|-143168397|6160556|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|snapshot.sparse.img|10|475782429|53428308|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|update_package.zip|10|-678170682|227494562|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|vr.inf|10|2034194806|58|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|vr.md5|10|-2044697631|41|1
2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system|vr.sha|10|-189486074|137|1
...
```
For each file we have its directory, name, CRC32 (as signed 32bit int) and size. I quickly figured out that the most interesting file is `2018_20_Tucson_EU\DAUDIOPLUS_M\eur\tlfl\update\system\update_package.zip` which is an encrypted zip:

[{{% fluid_img class="pure-u-1-1" src="/images/update_package.png" %}}](/images/update_package.png)

I tried to unzip using the password discovered by greenluigi1 (`ahqltmTkrhk2018@@`) but it didn't work.

Cracking the zip password
---
I decided to try the same cracking tool as greenluigi1 which is [bkcrack](https://github.com/kimci86/bkcrack). It implements a known plaintext attack discovered by Eli Biham and Paul C. Kocher. My encrypted zip contains two zip files. So what would be a good known plaintext candidate? The ZIP file header of course! This is how it looks like:

[{{% fluid_img class="pure-u-1-1" src="/images/zip-header.png" %}}](/images/zip-header.png)

Another significant advantage is that both zip files in `update_package.zip` are stored without compression (there is no point to compress files which are already compressed). This can be verified with the following command:

```shell
$ bkcrack -L update_package.zip
bkcrack 1.5.0 - 2022-07-07
Archive: update_package.zip
Index Encryption Compression CRC32    Uncompressed  Packed size Name
----- ---------- ----------- -------- ------------ ------------ ----------------
    0 ZipCrypto  Store       fbe57f09    227492981    227492993 update.zip
    1 ZipCrypto  Store       b4be54fe         1203         1215 otacerts.zip
```

Now we just need to guess 12 bytes of either `update.zip` or `otacerts.zip` file header to perform the attack. There are not too many options for the first 10 bytes. The first 4 bytes are fixed (`504b0304`). The version bytes are either `1400` or `0a00` in most of the cases. Assuming the nested zips are not encrypted, the general purpose bit flag should be `0000`. The compression method is either `0000` (store) or `0800` (deflate). So we have the following variants for the first 10 bytes:
```text
plain1.bin: 504b 0304 1400 0000 0000
plain2.bin: 504b 0304 1400 0000 0800
plain3.bin: 504b 0304 0a00 0000 0000
plain4.bin: 504b 0304 0a00 0000 0800
```
We need two more bytes and the candidates are either file modification time or file modification date. The time is stored in seconds divided by 2 precision which gives us 24\*60\*30=43200 possibilities. Assuming the file modification happened in the last 5 years, we have 5\*365=1825 possibilities for the date field. Clearly, it's more efficient to brute force the date field. However, a single run of this attack takes 30min on my laptop which makes brute force unfeasible. It's time for another wild guess. Both `update.zip` and `otacerts.zip` have "21 April 2022" as file modification date. Let's assume the modification date of the first file entry in `otacerts.zip` is also "21 April 2022". This date is encoded in MS-DOS format as `9554`. Now we have 12 bytes of plain text but they are not contiguous because we have skipped the file modification time. Fortunately, `bkcrack` supports specifying an offset with the `-x` option. The first run of the attack with `plain1.bin` failed. However, with `plain2.bin` I have successfully recovered the keys:

```shell
$ bkcrack -C update_package.zip -c otacerts.zip -p plain2.bin -x 12 9554
bkcrack 1.5.0 - 2022-07-07
[15:03:29] Z reduction using 3 bytes of known plaintext
100.0 % (3 / 3)
[15:03:29] Attack on 1677473 Z values at index 6
Keys: 850725d9 64202f01 143f9452
2.8 % (47031 / 1677473)
[15:04:05] Keys
850725d9 64202f01 143f9452
```

Extracting the zip files:

```shell
$ bkcrack -C update_package.zip -c update.zip -k 850725d9 64202f01 143f9452 -d update.zip
$ bkcrack -C update_package.zip -c otacerts.zip -k 850725d9 64202f01 143f9452 -d otacerts.zip
```

`otacerts.zip` contains a single X509 certificate modifed on 21 April 2022 (yes, I was very lucky!):

[{{% fluid_img class="pure-u-1-1" src="/images/otacerts.zip.png" %}}](/images/otacerts.zip.png)

`update.zip` contains all of the interesting stuff:

[{{% fluid_img class="pure-u-1-1" src="/images/update.zip.png" %}}](/images/update.zip.png)

Reversing the firmware
---
Apparently, my car is running Android and `system.ext4` is the root file system.
Let's mount and look inside:

```shell
$ mkdir /tmp/car
$ sudo mount -t ext4 -o loop system.ext4 /tmp/car
$ ls -la /tmp/car
total 140
drwxr-xr-x 14 root root  4096 Jan  1  1970 .
drwxrwxrwt 25 root root 65536 Aug 11 15:40 ..
drwxr-xr-x  2 root root  4096 Apr 21 10:50 app
drwxr-xr-x  2 root 2000  4096 Apr 21 10:46 bin
-rw-r--r--  1 root root  6020 Apr 21 10:26 build.prop
drwxr-xr-x 11 root root  4096 Apr 21 10:50 etc
drwxr-xr-x  2 root root  4096 Apr 21 10:35 fonts
drwxr-xr-x  2 root root  4096 Apr 21 10:50 framework
-rw-r--r--  1 root root  1912 Apr 21 10:35 key_3000000.psr
-rw-r--r--  1 root root  1913 Apr 21 10:35 key_921600.psr
drwxr-xr-x  8 root root  8192 Apr 21 10:46 lib
drwx------  2 root root  4096 Jan  1  1970 lost+found
drwxr-xr-x  4 root root  4096 Apr 21 10:35 media
drwxr-xr-x  7 root root  4096 Apr 21 10:38 usr
drwxr-xr-x  4 root 2000  4096 Apr 21 10:35 vendor
drwxr-xr-x  2 root root  4096 Apr 21 10:35 wifi
drwxr-xr-x  2 root 2000  4096 Apr 21 10:45 xbin
```
Pretty much all of the applications are the in `/app` folder. Each app has both .apk and .odex, e.g:
```shell
$ ls -l /tmp/car/app/HKMC_EngineerMode.*
-rw-r--r-- 1 root root 2124079 Apr 21 10:50 /tmp/car/app/HKMC_EngineerMode.apk
-rw-r--r-- 1 root root 1630136 Apr 21 10:50 /tmp/car/app/HKMC_EngineerMode.odex
```
I have used [baksmali](https://github.com/JesusFreke/smali) to convert odex files to smali and [jadx](https://github.com/skylot/jadx) to decompile smali to Java.
Let's look into the apps for Engineering Mode and then software updates.

```shell
$ java -jar baksmali-2.5.2.jar deodex -a 17 -d /tmp/car/framework -o engmode /tmp/car/app/HKMC_EngineerMode.odex
$ java -jar baksmali-2.5.2.jar deodex -a 17 -d /tmp/car/framework -o swupdate /tmp/car/app/HKMC_SwUpdate.odex
```

Engineering mode password
---
The password for the engineering mode is obtained in the `getKeyString()` method of `com.hkmc.system.app.engineering.ime.InputPasswordExt`.
It takes the last digit of the current year and then it does a resource lookup:
[{{% fluid_img class="pure-u-1-1" src="/images/engmode-pwd1.png" %}}](/images/engmode-pwd1.png)

These are the corresponding resource strings:

[{{% fluid_img class="pure-u-1-1" src="/images/engmode-pwd2.png" %}}](/images/engmode-pwd2.png)

This year's password is __2702__. I tried it on the car and it worked:

[{{% fluid_img class="pure-u-1-1" src="/images/engmode-unlocked.jpg" %}}](/images/engmode-unlocked.jpg)

Sofware update password
---
Even though I can extract firmware updates with the keys derived with `bkcrack`, I was curious to find out the password for `update_package.zip`.
A simple grep for `"update_package.zip"` leads to the `getPassword()` method of `com.hkmc.system.app.swupdate.ImageFileCopy`:

[{{% fluid_img class="pure-u-1-1" src="/images/update-pwd.png" %}}](/images/update-pwd.png)

The static password "+Ekfrl51Qkshsk#@zpdhkdWkd~-f" didn't work. In my case the password was derived from the concatenation of the following system properties:
```text
ro.product.model=daudioplus
ro.product.brand=hyundai
ro.product.name=tlfl_eu
ro.product.device=daudiopluslow_tlfl_eu
ro.product.board=daudio
ro.product.cpu.abi=armeabi-v7a
ro.product.cpu.abi2=armeabi
ro.product.manufacturer=mobis
ro.product.locale.language=en
ro.product.locale.region=GB
```

The `SHA512Checking()` method computes SHA512 digest of the given string and returns the result as uppercase hex string.
Doing this twice and taking a substring:

```java
SHA512Checking(SHA512Checking(tmp1)).substring(10, 38) = "15DAA85C8D44B3979CD152A387F6"
```

The password is __15DAA85C8D44B3979CD152A387F6__

Future work
---
I am pretty sure that I can create a firmware update with backdoor and get a root shell.
But there is a small chance for something to go wrong and brick my car, so I am not going to do this for now :)
I have some ideas how to exploit the kernel with the information from the engineering mode, so I will try this first.
The ultimate goal, of course, is to play Doom on my car screen.
