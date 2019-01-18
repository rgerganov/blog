+++
topics = []
tags = []
draft = false
date = "2017-01-29T13:27:39+02:00"
title = "Cloning RFID cards"
description = ""

+++

This post summarizes my experience with cloning RFID cards that I am using on daily basis.
There is nothing new here, just a summary of well-known hacks that I found on the internet.


### Corporate badge

At work we use 125KHz passive RFID badges which are easy to clone.
Each badge has unique ID, so the first step is to read this ID.
I have been using this DIY [reader](http://playground.arduino.cc/Main/DIYRFIDReader) based on an Arduino:

[{{% fluid_img class="pure-u-1-1" src="/images/rfid-reader-small.jpg" %}}](/images/rfid-reader.jpg "rfid-reader")

By the way, I had to make some slight [changes](https://gist.github.com/rgerganov/c8cec1f2c498c1e0786084bfdc1240b7) to the firware to make it work.

Once you get the ID, you can "program" it on an ATtiny85 microcontroller as described [here](http://scanlime.org/2008/09/using-an-avr-as-an-rfid-tag/).
This is how my assembled clone looks like:

[{{% fluid_img class="pure-u-1-1" src="/images/rfid-badge-small.jpg" %}}](/images/rfid-badge.jpg "rfid-badge")

### Residential entry

The building where I live (and many other buildings) has a front door which is opened with an NFC tag. I found that it is Mifare Classic 1k tag
which has been cracked long time ago. Actually, it turned out that I don't have to crack anything to create a clone because the tag had default keys.
They were relying only on the fact that each tag has unique ID and the door opens when pre-recorded ID is shown. 
Some fine people in China are selling tags that you can program with whatever ID you want, so creating is a clone a simple matter of running `nfc-mfsetuid`:

<script type="text/javascript" src="https://asciinema.org/a/14.js" id="asciicast-99020" async></script>


### Public transport

Turns out that Mifare Classic 1k is also being used for the public transport in Sofia. I managed to crack one of the old cards that I have -- it has been
using a combination of default keys and "SofiaM": 

<script type="text/javascript" src="https://asciinema.org/a/14.js" id="asciicast-a1fjcus3zcgkxpvh3i72txy91" async></script>

However, some of the new cards for the subway manage to resist my cracking attempts.
I guess they are using something more secure which is running an emulation of Mifare Classic.

UPDATE 03/15/2017: No, they are not. Keys have been disclosed [here](https://www.facebook.com/ilf.fb/posts/10158277574290527).

