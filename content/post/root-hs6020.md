+++
title = "Rooting HS6020 IPTV STB"
date = "2019-11-05T17:12:08+02:00"
tags = []
topics = []
description = ""
+++

One of my co-workers brought a very old IPTV set-top-box in the office. 
The box was distributed by one of the very first IPTV providers in Sofia -- Megalan.
I remember using one of those back in 2009. 
We decided to see how difficult is to root 10+ years old hardware and the fun began!

The box is running both telnet and ssh but there is no default password.
The next thing we tried, of course, is connecting a serial console.
With some trial and error we found the pins on the PCB and used an Arduino board as 
USB-to-serial converter:

[{{% fluid_img class="pure-u-1-1" src="/images/hs6020-small.jpg" alt="ac2729" %}}](/images/hs6020.jpg "hs6020")

```
screen /dev/ttyACM1 115200
```


```
Common Firmware Environment (CFE) version 1.1 for BCM97401C1, (Little Endian Mode)
Build Date: Wed Jan 23 13:31:55     2008 (fangyu@hansunte-d6fcaf)
Copyright (C) 2000-2007 Broadcom Corporation.

Initializing Arena.
Initializing Devices.
env loaded
Initializing Devices finish.
CPU speed: 297MHz
Total memory: 0x8000000 bytes (128MB)

...

(none) login: 
```


The device boots and we get a nice login prompt. 
It's a Broadcom SoC and uses something called Common Firmware Environment (CFE) to boot the linux kernel. 
After a few google searches, I found that CFE can be interrupted with Ctrl+C which gives us something like a bootloader shell:

```
Automatic startup canceled via Ctrl-C
CFE> printenv
Variable Name        Value
-------------------- --------------------------------------------------
        BOOT_CONSOLE uart0
         ETH0_HWADDR 00:18:95:10:10:6B
             STARTUP boot -z -elf flash0.kernel0: 'root=/dev/mtdblock0 rootfstype=jffs2 mem=64m '
         CFE_VERSION 1.1.0
       CFE_BOARDNAME BCM97401C1
      CFE_MEMORYSIZE 128
*** command status = 0
```

The printenv command reveals the `STARTUP` variable which shows the exact boot command.
The last parameter of the boot command looks like a kernel cmdline.
Let's try and run the shell instead of the init process:

```
CFE> boot -z -elf flash0.kernel0: 'root=/dev/mtdblock0 rootfstype=jffs2 mem=64m init=/bin/sh'
```

The device boots and we have a root shell :)

```shell
# cat /etc/shadow
root:BGOfOe8QJYuyA:0:0:root:/root:/bin/sh
bin:*:1:1:bin:/bin:/dev/null
daemon:*:2:2:daemon:/sbin:/dev/null
adm:*:3:4:adm:/var/tmp:/dev/null
ftp:*:14:50:FTP User:/var/tmp:/dev/null
nobody:*:99:99:Nobody:/:/dev/null
rpcuser:x:29:29:RPC Service User:/var/tmp:/dev/null
nfsnobody:x:65534:65534:Anonymous NFS User:/var/tmp:/dev/null
```

The hash for the root password is `BGOfOe8QJYuyA` which is a DES hash.
I tried a quick dictionary attack on this hash without success.
This doesn't matter because the file system is writable and we can replace the hash with whatever we want.
After changing the hash and rebooting, we can login on telnet:

```shell
telnet 10.10.20.244 
Trying 10.10.20.244...
Connected to 10.10.20.244.
Escape character is '^]'.
(none) login: root
Password: 
# whoami
root
# cat /proc/cpuinfo
system type             : BCM97xxx Settop Platform
processor               : 0
cpu model               : Brcm7401 V0.0
cpu MHz                 : 295.93
BogoMIPS                : 295.93    ( udelay_val : 147968  HZ = 1000 )
wait instruction        : yes
microsecond timers      : yes
tlb_entries             : 32
extra interrupt vector  : yes
hardware watchpoint     : no
ASEs implemented        :
VCED exceptions         : not available
VCEI exceptions         : not available
RAC setting             : I/D-RAC enabled
unaligned access        : 6
# uname -a
Linux (none) 2.6.12-5.0-brcmstb #45 Sat Dec 6 21:52:44 EST 2008 7401c0 unknown
```
