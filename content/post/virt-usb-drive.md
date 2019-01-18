+++
date = "2017-03-22T10:21:41+02:00"
description = ""
draft = false
tags = []
title = "Virtual USB drive"
topics = []

+++

This is a PoC for something I call "virtual usb drive". The drive is created on Linux using the
[MSG](https://www.kernel.org/doc/Documentation/usb/mass-storage.txt) kernel module.
Then it can be attached to VMs running on VMware vSphere using VMRC 5.5. I think it's pretty cool hack, so I decided
to share it. 

**DISCLAIMER**: This is unofficial and unsupported, use it at your own risk.

Here is a demo:

{{< youtube RSxUyQDnd2w >}}

You can find the supporting scripts [here](https://gist.github.com/rgerganov/df442733da54ad104bf03d81355a845e).

