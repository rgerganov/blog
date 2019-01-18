+++
date = "2019-01-19T10:49:37+02:00"
title = "Controlling my Air Purifier"

+++

The air pollution in Sofia is really high during the winter, so I decided to buy an air
purifier for my home. After some short research, I bought Philips AC2729:

[{{% fluid_img class="pure-u-1-1" src="/images/ac2729-small.jpg" alt="ac2729" %}}](/images/ac2729.jpg "ac2729")

it can purify and humidify the air at the same time, show particulate matter (PM 2.5) /
humidity, and features quiet sleep mode. It also has a Wi-Fi interface which allows
remote control with a mobile app. Being able to control the device with my phone sounds
pretty cool but after reading so many stories about the so called
[Internet of Shit](https://twitter.com/internetofshit), I thought it'd be a good idea to
inspect what kind of data this thing sends and ultimately create my own tool to control
it. It'd be a fun side project as well :)

Network analysis
---
The first step of course is capturing the network traffic that comes in and goes out from
the device. There are several ways to do this with a PC and good Wi-Fi adapter but I took
advantage of the built-in
[packet sniffer](https://wiki.mikrotik.com/wiki/Manual:Tools/Packet_Sniffer) that my
router has and recorded the traffic during the initial setup of the purifier and some
interactions with the [mobile app](https://play.google.com/store/apps/details?id=com.freshideas.airindex).
Then I opened the recorded .pcap file in Wireshark and quickly realized couple of things: 

 * the device is sending data to a Philips server (www.ecdinterface.philips.com),
 no surprise here ...
 * the device runs an HTTP server and is controlled via some REST API from the mobile app
 * it uses plain HTTP with some custom encryption for the request/response body

I can easily disable the phone-home functionality by putting a firewall rule in my router
and limit all traffic to my local network. But I also want to understand how the device
is controlled and for this I need to decrypt the network traffic between the purifier and
the mobile app.

Let's take a closer look to the initial interaction between those two
parties:

```
PUT /di/v1/products/0/security HTTP/1.1
content-type: application/json
connection: close
User-Agent: Dalvik/2.1.0 (Linux; U; Android 9; Pixel XL Build/PQ1A.181205.002.A1)
Host: 192.168.0.17
Accept-Encoding: gzip
Content-Length: 269

{"diffie":"4FB7FC5543219711B7144FDA72E4A2..."}

HTTP/1.1 200
Content-Length: 343
Content-Type: application/json
Connection: Close

{"hellman":"4a5008cbc6e4523f27e...",
 "key":"9d90df8e281855117467e09fa75de7aa60097c16657a5cc04b094e7dbb4b0518"}

GET /di/v1/products/1/device HTTP/1.1
content-type: application/json
connection: close
User-Agent: Dalvik/2.1.0 (Linux; U; Android 9; Pixel XL Build/PQ1A.181205.002.A1)
Host: 192.168.0.17
Accept-Encoding: gzip

HTTP/1.1 200
Content-Length: 128
Content-Type: application/octet-stream
Content-Transfer-Encoding: base64
Connection: Close

SwZyJzgE66DpOjL6nshHS/xjF9VHxL7Fg0XjHDmDtNhBBBMC4FeuJzBQiQx8QFcIlyoOTaEPUq
QqDmN1f2we0MRjSjyt4aEenNlvtvj/vCa+e15btQcZYWcqTHvpr1Pb

POST /di/v1/products/1/air HTTP/1.1
content-type: application/json
connection: close
User-Agent: Dalvik/2.1.0 (Linux; U; Android 9; Pixel XL Build/PQ1A.181205.002.A1)
Host: 192.168.0.17
Accept-Encoding: gzip
Content-Length: 65

Mt32GhsP00A6cDdHf+8vBMMKsJU/rZsD7onyiTjNDxPlRhepKspv/lw9GKhhYc6O

HTTP/1.1 200
Content-Length: 256
Content-Type: application/octet-stream
Content-Transfer-Encoding: base64
Connection: Close

NbV7wqswB4p3/yV6KSBN92c9sLy3tOjbUaCWqfddss1oIUNGl3EPacivoqROasGPS9x8VRwRIV734jQ0Q
Amm/0OehccirajReq0/yEcwV7jo+gbkEIaIUNdVE/XccXptR3VTsO3W/7ge5U9wM7NM9jAz7BgvkKoEtjg
7+3Rs7M9lgiTpcDr85b5NMo/tIpzeKM7+CMtmI9toOaDmnjxqoJO1JhpAk7VP5lCO5m/iFfFWPgeWYSlvw
YK6SGVqzqlP
```

The first request to `/di/v1/products/0/security` is plain text and it seems to be a
[Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) key
exchange (truncated by me for readability). The following requests are base64 encoded and
decoding them doesn't produce anything meaningful. So it looks like a secret key is
exchanged at the beginning and then used to encrypt further communication.
There is only one way to verify this theory -- reverse engineering the mobile app!

Reversing the original app
---

The mobile app is Air Matters and I downloaded its Android version which is an apk file.
Then I used the [jadx](https://github.com/skylot/jadx) tool to decompile it. With a few
greps over the decompiled sources, I found the relevant encryption/decryption classes:

```
com/philips/cdp2/commlib/lan/communication/ExchangeKeyRequest.java
com/philips/cdp/dicommclient/security/EncryptionUtil.java
com/philips/cdp/dicommclient/security/ByteUtil.java
```

With this I have been able to recover the following steps:

1. The app and the device establish a shared secret using the Diffie-Hellman algorithm
2. The device generates a session key and encrypts it with AES using the shared secret
from step 1 as a key. The encrypted session key is set in the `key` property of the
response message from `PUT /di/v1/products/0/security`
3. The session key from step 2 is used to encrypt/decrypt with AES all further requests
and responses

I also found the `G` and `P` values used for Diffie-Hellman in `ByteUtil.java`:

```
static final String GVALUE = "A4D1CBD5C3FD34126765A442EFB99905F8104DD258AC507FD6406CFF14266D31266FEA1E5C41564B777E690F5504F213160217B4B01B886A5E91547F9E2749F4D7FBD7D3B9A92EE1909D0D2263F80A76A6A24C087A091F531DBF0A0169B6A28AD662A4D18E73AFA32D779D5918D08BC8858F4DCEF97C2A24855E6EEB22B3B2E5";
static final String PVALUE = "B10B8F96A080E01DDE92DE5EAE5D54EC52C99FBCFB06A3C69A6A9DCA52D23B616073E28675A23D189838EF1E2EE652C013ECB4AEA906112324975C3CD49B83BFACCBDD7D90C4BD7098488E9C219A73724EFFD6FAE5644738FAA31A4FF55BCCC0A151AF5F0DC8B4BD45BF37DF365C1A65E68CFDA76D4DA708DF1FB2BC2E4A4371";
```

However, this is not enough to find the shared secret which is being established.
In fact, `G` and `P` are public values. Cracking Diffie-Hellman boils down to solving the
discrete logarithm problem which is to find a value `"a"` such that `G^a mod P=A`.
We already have G and P from the decompiled source and we also have `A` which corresponds
to the `"diffie"` property in the network request. The shared secret is equal to
`B^a mod P` where `B` corresponds to the `"hellman"` property in the network response.
The value `"a"` is by definition a big random number which makes solving `G^a mod P=A` very
hard. Let's see how this random exponent is generated in `ByteUtil.java`:

```
public static String generateRandomNum() {
    return String.valueOf((new SecureRandom().nextInt(2147483546) + 1) + 101);
}
```

It seems this random number is only 31 bits. It also seems that the person who wrote this
didn't know what he is doing :) Solving the discrete logarithm problem when `a` is only
31 bits turns out to be not hard at all. In fact, it can be solved for less than a second
using the [Baby-step Giant-step](https://en.wikipedia.org/wiki/Baby-step_giant-step)
algorithm. Check out my implementation at https://github.com/rgerganov/dlp.

Open source client
---
After finding the session key and decrypting the entire traffic, I have been able to
implement my own fully-featured command line
[client](https://github.com/rgerganov/py-air-control). Check this out:

```
$ ./airctrl.py 192.168.0.17
[pwr]   Power: ON
[pm25]  PM25: 4
[rh]    Humidity: 32
[rhset] Target humidity: 60
[iaql]  Allergen index: 1
[temp]  Temperature: 22
[func]  Function: Purification & Humidification
[mode]  Mode: M
[om]    Fan speed: 2
[aqil]  Light brightness: 100
[wl]    Water level: 100
[cl]    Child lock: False
```

As an added bonus, it also shows the current temperature which is missing in the original
mobile app :)

Future work
---

As I said the purifier has a phone-home functionality which is using another form of
custom encryption but this time implemented in a native library.  Reversing this part
will take more time but something tells me it may allow doing some crazy stuff ;)

Stay tuned!
