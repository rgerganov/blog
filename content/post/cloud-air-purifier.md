+++
title = "Controlling purifiers over the internet"
date = "2019-11-30T11:04:38+02:00"
tags = []
topics = []
description = ""
+++
[Last time](/post/ctrl-air-purifier/) I reverse engineered how [AirMatters](https://play.google.com/store/apps/details?id=com.freshideas.airindex) controls air purifiers which are in the same local network.
This time we will look into how AirMatters controls devices using the Philips cloud and how secure it is.

Network analysis
---
The network communication is over SSL, so the first step is to bypass the SSL verification in AirMatters.
The application loads its CA certificates from `/resources/assets` and tries to establish a trust chain to them.
My favourite tool for sniffing SSL traffic is `mitmproxy`, so I simply took `~/.mitmproxy/mitmproxy-ca-cert.cer` from my machine and re-packaged AirMatters with it.

This is what I found after capturing the network traffic:

1. Calling the cloud services requires logging with `client_id` and `client_key` first.
2. AirMatters is using hardcoded credentials to provision a new account and obtain `client_id` and `client_key`
3. Every request contains an `Authorization` HTTP header which is verified by the server
4. You need to pair the `client_id` with the purifier's `device_id` and then you can control it over the internet. The pairing is done by sending a special pairing request to the device and another pairing request to the corresponding cloud service.

The problem with this pairing mechanism is that it is completely silent and doesn't require any physical interaction with the device. This means that an attacker needs one time network access to your device to pwn it forever. As far as I can tell, there is no way to "unpair" a device or to see how many accounts your device is paired with.

The only missing piece in the above is how the `Authorization` HTTP header is calculated. This is how it looks like:
```
Authorization: CBAuth Type="SSO", Client="000000fff10af31f", RequestNr="2", Nonce="kj3PUx56Cpo3UdcIiICoPQ==", SSOToken="kcK0TgzC..", Authentication="XAR9WB443KTDT0A7pNUQZA=="
```
We get `SSOToken` and `Nonce` from the response of the login request. The `Authentication` is 16 bytes base64 encoded value which is a calculated in `libICPClient.so`.

Reversing `libICPClient.so`
---
I used [Ghidra](https://ghidra-sre.org/) to reverse engineer this native library and find how the authentication bytes are computed.
This is how the relevant function looks like after being decompiled:

{{% fluid_img class="pure-u-1-1" src="/images/key1.png" caption="ghidra-key1" %}}

It does SHA1 hash of client_id, host, sso_key, the request number, nonce and then it takes the first 16 bytes of the digest:

{{% fluid_img class="pure-u-1-1" src="/images/key2.png" caption="ghidra-key2" %}}

And this is how I implemented it in Python:

```python
m = hashlib.sha1()
m.update(b'\x05')
m.update(client_id.encode('ascii'))
m.update(host.encode('ascii'))
m.update(base64.b64decode(sso_key))
m.update(struct.pack('<I', request_num))
m.update(base64.b64decode(nonce))
key = m.digest()[:16]
auth = base64.b64encode(key).decode('ascii')
```

Open source client
---
With this I put together `cloudctrl` which can be used together with `airctrl` to provision accounts, pair and control purifiers over the internet.
Check out the [README](https://github.com/rgerganov/py-air-control/blob/master/README.md) of my project for more information.

Disclosure
---
After publishing my previous post, I have been contacted by Philips and they told me about their [coordinated vulnerability disclosure](https://www.philips.com/a-w/security/coordinated-vulnerability-disclosure.html) process.
So this time I coordinated my new findings with them before making it public. As a result they put me into their [hall of honors](https://www.philips.com/a-w/security/coordinated-vulnerability-disclosure/hall-of-honors.html) :)

