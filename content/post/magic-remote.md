+++
title = "Reversing MAGic Remote"
date = "2025-01-19T13:31:54+02:00"
tags = []
topics = []
description = ""
+++

If you have a [MAG](https://www.infomir.eu/eng/products/iptv-stb/) set-top box, you may know there is a mobile app called ["MAGic Remote"](https://play.google.com/store/apps/details?id=com.infomir.magicRemote&hl=en) which turns your phone into a remote controller. It looks like this:

[{{% fluid_img class="pure-u-1-1" src="/images/magic-remote-small.png" %}}](/images/magic-remote.png "MAGic Remote")

Allowing remote control over the network is great but there is no public information on the protocol being used.
MAGic Remote is also paid and I don't quite understand why people should be giving money for something so simple.
That was my motivation to reverse engineer the protocol and create an open-source client.

## Network analysis

MAGic Remote discovers set-top boxes in the local network using [zeroconf](https://en.wikipedia.org/wiki/Zero-configuration_networking), looking for `_infomir_mobile_rc_service._tcp` services. Then it sends a pairing request to the set-top box which shows a pairing code on the TV. The user has to enter this code in the app to pair the devices:

{{% fluid_img class="pure-u-1-1" src="/images/magic-remote-pair.png" caption="Pairing code" %}}

The first step is to capture the network traffic during the pairing process. Packets in red are sent by the app and packets in blue are sent by the set-top box:

{{% fluid_img class="pure-u-1-1" src="/images/magic-remote-pcap.png" caption="Pairing packet dump" %}}

They seem to be using a request-response protocol over TCP and all messages have the following structure:

```
[6 bytes]  - message header
[32 bytes] - command
[...]      - body
```

The command is always sent in plain text (e.g. `pairing-reqpairing-reqpairing-re`, `pairing-complete-reqpairing-comp`, etc.),
while the body is sometimes in binary and sometimes in plain text.

The pairing code doesn't seem to be send in plain text anywhere, so I'd expect it is used as mutual authentication between the app and the set-top box.

## Static analysis

The go-to app for decompiling and static analysis of Android apps is [jadx](https://github.com/skylot/jadx).
A quck search for encryption-related classes reveals the following class in the `I5` package:

{{% fluid_img class="pure-u-1-1" src="/images/magic-remote-jadx.png" caption="Decompiled class" %}}

This method takes a password, appends a fixed suffix to it, hashes it with SHA1 and uses the first 16 bytes as the key for AES encryption.
The IV is hardcoded and the encryption mode is CFB.

## Dynamic analysis

I used [Frida](https://frida.re/) to trace the calls to this method and capture the arguments and return values:

```bash
$ frida-ps -Ua
  PID  Name               Identifier
-----  -----------------  ---------------------------------------
10610  Contacts           com.google.android.contacts
 2623  Google             com.google.android.googlequicksearchbox
 3465  Google Play Store  com.android.vending
10749  Google Wallet      com.google.android.apps.walletnfcrel
 8362  Keep Notes         com.google.android.keep
 5400  MAGic Remote       com.infomir.magicRemote
10090  Meet               com.google.android.apps.tachyon
10488  Messages           com.google.android.apps.messaging
 9754  Photos             com.google.android.apps.photos
 9538  YouTube            com.google.android.youtube
$ frida-trace -U -j 'I5.a!*' 5400
Instrumenting...
a.$init: Loaded handler at "/opt/src/frida/__handlers__/I5.a/_init.js"
a.a: Loaded handler at "/opt/src/frida/__handlers__/I5.a/a.js"
a.b: Loaded handler at "/opt/src/frida/__handlers__/I5.a/b.js"
Started tracing 3 functions. Web UI available at http://localhost:38275/
           /* TID 0x1566 */
  9961 ms  a.b("259518", [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,125])
  9974 ms  <= [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-16]
  9984 ms  a.a("259518", [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-16])
  9986 ms  <= [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,125]
  9989 ms  a.b("259518", [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,44,34,100,101,118,95,100,101,115,99,114,34,58,34,71,111,111,103,108,101,32,80,105,120,101,108,32,88,76,34,44,34,114,99,95,99,111,100,101,34,58,49,57,48,125])
  9992 ms  <= [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-95,52,76,-100,-105,-121,7,94,-32,-84,-111,-51,-60,-15,74,-69,9,-2,8,91,26,-3,59,111,-83,-77,-96,118,19,0,113,-13,-109,68,-5,69,120,-44,-77,-38,-82,-31,103,-102,-114]
           /* TID 0x1538 */
 22182 ms  a.b("259518", [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,125])
 22185 ms  <= [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-16]
 22254 ms  a.a("259518", [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-16])
 22256 ms  <= [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,125]
 22259 ms  a.b("259518", [123,34,100,101,118,95,105,100,34,58,34,102,97,101,97,99,57,101,99,52,49,99,50,102,54,53,50,34,44,34,100,101,118,95,100,101,115,99,114,34,58,34,71,111,111,103,108,101,32,80,105,120,101,108,32,88,76,34,44,34,114,99,95,99,111,100,101,34,58,49,57,49,125])
 22262 ms  <= [16,13,116,-62,-19,82,-85,91,-21,9,42,31,-107,-71,29,51,-23,84,1,37,96,-23,-30,7,9,50,38,-55,-95,52,76,-100,-105,-121,7,94,-32,-84,-111,-51,-60,-15,74,-69,9,-2,8,91,26,-3,59,111,-83,-77,-96,118,19,0,113,-13,-109,68,-5,69,120,-44,-77,-38,-82,-31,103,-101,-114]
           /* TID 0x300d */
```

Apparently, they use the pairing code as a first argument to the encryption method and the second argument is a JSON string.
Having some input and output data, we can easily reconstruct the encryption/decryption method in Python:

```python
def encrypt(pwd, data):
    pwd_bytes = pwd.encode('utf-8')
    suffix = [8, 56, -102, -124, 29, -75, -45, 74]
    to_hash = pwd_bytes + to_uint8(suffix)
    h = SHA1.new()
    h.update(to_hash)
    key = h.digest()[:16]
    iv = to_uint8([18, 111, -15, 33, 102, 71, -112, 109, -64, -23, 6, -103, -76, 99, -34, 101])
    # same as Cipher.getInstance("AES/CFB8/NoPadding") in Java
    cipher = AES.new(key, AES.MODE_CFB, iv, segment_size=128)
    return cipher.encrypt(data)
```

With this we can decrypt all messages from the network dump and see the plain text commands and bodies.

## Open-source client

I put together a simple [Python script](https://github.com/rgerganov/magic-remote) which implements the entire functionality of the MAGic Remote app.
With this you have full control over your set-top box and you can integrate it with other home automation systems.