+++
title = "How many people are around"
date = "2020-05-26T17:41:25+03:00"
tags = []
topics = []
description = "COVID-19 edition"
+++

There is a nice open-source project [howmanypeoplearearound](https://github.com/schollz/howmanypeoplearearound)
that counts the number of people around by sniffing WiFi probe requests sent from mobile phones.

Well, now we have another method to do the same by exploiting the [contact tracing](https://en.wikipedia.org/wiki/Exposure_Notification) functionality which is being
added to iOS and Android. Cell phones are using Bluetooth Low Energy to transmit ephemeral IDs to nearby
devices in order to discover encounters with other people. These IDs and the Bluetooth MAC changes every 15-20
minutes to prevent tracking of users. However, this time is enough to answer the question "how many people are around?".
We can also differentiate contact tracing messages from other messages by the service UUID.
The UUID used by [Google and Apple](https://blog.google/documents/70/Exposure_Notification_-_Bluetooth_Specification_v1.2.2.pdf) is `0xFD6F`.

Here is my PoC running on Linux which uses [tshark](https://www.wireshark.org/docs/man-pages/tshark.html).
First we dump BTLE advertisements to a file for a couple of minutes:
```bash
$ sudo hcitool lescan --duplicates &
$ tshark -i bluetooth0 -w /tmp/btcap
# leave this for 5-10 min.
```

Then we filter by service UUID and print the MAC address:
```bash
$ tshark -r /tmp/btcap -T fields -e bthci_evt.bd_addr -Y "btcommon.eir_ad.entry.uuid_16 == 0xfd6f"
5c:32:95:32:93:86
5c:32:95:32:93:86
5c:32:95:32:93:86
5c:32:95:32:93:86
74:57:03:25:71:b9
74:57:03:25:71:b9
74:57:03:25:71:b9
74:57:03:25:71:b9
```

From this output we can say that we have either one device which changed its MAC address during the scan or two devices around.
