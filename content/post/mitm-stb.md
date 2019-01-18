+++
date = "2016-09-27T11:19:21+03:00"
title = "MitM'ing my STB"

+++

My ISP is offering IPTV with a set-top-box (STB) device which is connected to
the provider network and the TV itself:

{{% fluid_img class="pure-u-1-1" src="/images/stb-diagram.png" caption="stb-diagram" %}}

The only "user interface" for the STB is the remote control.  I was curious to
find out if the traffic between the STB and the provider is encrypted, so I
decided to see what goes on the wire.  The easiest way to do this is to create
an ethernet bridge between the provider and the STB and then capture the
traffic.  I used my MacBook with a second USB Ethernet adapter. In OSX, an
ethernet bridge is created like this:

```
    ifconfig bridge1 create
    ifconfig bridge1 up
    ifconfig bridge1 addm en4 addm en5
    sysctl -w net.inet.ip.forwarding=1
```

I used Wireshark to capture the traffic and to my surprise there was no
encryption at all. They have been using
[RTSP](https://en.wikipedia.org/wiki/Real_Time_Streaming_Protocol) to switch
between TV channels:

{{% fluid_img class="pure-u-1-1" src="/images/wireshark-stb.png" caption="wireshark capture" %}}

Every channel has unique ID like `ch12032017033910402807`. The ISP is also
offering some paid channels (like HBO) which I am not subscribed to.  However, I
managed to find the complete list of channel IDs in the traffic dump, including
the ones that I am not subscribed to.  In other words, the entire security for
the paid channels is done on the client side (by the STB) instead on the server
side.  And you can watch every channel as long as you know its ID.

At this point my first idea was to replace the STB with my own computer which
can talk their protocol and play the entire set of channels.  However, that
would be a lot of work which is not fun doing. A better solution would be to
create a man-in-the-middle between the STB and the provider which can modify the
traffic and thus substitute some crappy channels with the paid channels. What a
great opportunity to use my old Raspberry Pi!

The classic MitM with ARP spoofing didn't work out because there was MAC filter
on the provider side.  Long story short, I created my own tool
[nfqsed](https://github.com/rgerganov/nfqsed) which can transparently modify
network traffic using a predefined set of substitution rules. I used my USB
Ethernet adapter to turn my Raspberry Pi into an ethernet bridge. The built-in
network adapter is connected to the provider and the USB adapter is connected
to the STB. This is how it looks like from the top:

[{{% fluid_img class="pure-u-1-1" src="/images/stb-rpi-small.jpg" %}}](/images/stb-rpi.jpg "stb-rpi")

Below are the relevant config files on the Raspberry Pi which setup the
networking and configure nfqsed to replace one TV channel with another:

#### /etc/network/interfaces
```
iface br0 inet static
    bridge_ports eth0 eth1
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8
    up iptables -A FORWARD -p tcp --destination-port 554 -j NFQUEUE --queue-num 0
```

#### /etc/rc.local
```
/home/pi/nfqsed/nfqsed -f /home/pi/rules.txt &
```

#### /home/pi/rules.txt
```
# Fen TV / HBO HD
/ch13121216533621227255/ch12092914401822894191
# Folklor TV / HBO Comedy HD
/ch11123010551095204329/ch12092914413728185379
# Tqnkov / Cinemax
/ch11123010550900907719/ch11123010551005188322
# DM Sat / Cinemax 2
/ch11123010550925360552/ch11123010551067094950
```

