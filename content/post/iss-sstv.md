+++
title = "ISS SSTV"
date = "2021-06-24T16:47:22+03:00"
tags = []
topics = []
description = ""
+++

Today I have successfully received and decoded an [SSTV](https://en.wikipedia.org/wiki/Slow-scan_television) transmission from [ISS](https://en.wikipedia.org/wiki/International_Space_Station).
This is the result:

{{% fluid_img class="pure-u-1-1" src="/images/iss-sstv.png" caption="iss-sstv" %}}

In this post I will give a quick summary of what I have been using and how it worked for me.

Hardware:
---
 * USRP B200 - this is a high-end SDR but RTL-SDR dongle should also be fine
 * Dipole antenna - I bought this from the [RTL-SDR store](https://www.rtl-sdr.com/using-our-new-dipole-antenna-kit/); each side of the dipole should be around 50cm to make it resonant to 145.800MHz which is the frequency of the SSTV transmission
 * Laptop

I don't have a base station, so I brought everything on this rooftop (the Baofeng radio is not used):
[{{% fluid_img class="pure-u-1-1" src="/images/iss-sstv-rig-small.jpg" %}}](/images/iss-sstv-rig.jpg "iss-sstv-rig")

Software:
---
 * [gpredict](http://gpredict.oz9aec.net/) - I have used this to track ISS, it gives a nice schedule of all upcoming passes
 * [gqrx](https://gqrx.dk/) - my favorite SDR program; tune to 145.800MHz FM and record the audio; this is my [recording](https://drive.google.com/file/d/1I5k2g2cRtz5jN0REE-S1kHAZ6z2RTBaB/view?usp=sharing)
 * [qsstv](http://users.telenet.be/on4qz/qsstv/index.html) - an open-source program for SSTV decoding; the user interface is pretty bad but it does the job

That's all, have fun!

73!
