+++
title = "Cloning RSA tokens with Frida"
date = "2019-12-12T10:13:20+02:00"
tags = []
topics = []
description = ""
+++
At work we are using [RSA SecurID](https://en.wikipedia.org/wiki/RSA_SecurID) for login to corporate services.
RSA provides software tokens which are mobile applications that generate one-time passwords that change every minute or so.
These applications are closed source and they use proprietary protocol to provision the token.
This basically means that you are vendor locked-in with something that relies on security through obscurity.

I wanted to have an open-source implementation of RSA SecurID and until now I have been using [stoken](https://github.com/cernekee/stoken)
together with [rsa_ct_kip](https://github.com/dlenski/rsa_ct_kip) for token provisioning. This has been working great until
my employer decided to switch to the new [RSA SecurID Authenticate](https://play.google.com/store/apps/details?id=com.rsa.via) app which
uses some different kind of token provisioning that doesn't work with `rsa_ct_kip`. At this point I have two options:

1. Reverse engineer the mobile app and find how the provisioning is done.
I already did this for the old [app](https://play.google.com/store/apps/details?id=com.rsa.securidapp)
and this was my contribution to the `rsa_ct_kip` project. 
2. Provision a token with the new app on a rooted phone and then "export" it from there.

Reversing the provisioning protocol will take time, so I decided to go with option 2 for the short term. 
I already know that RSA tokens need two things to be provisioned -- serial number and root seed (16 bytes).
Once I get those, I can continue using `stoken` on all of my devices.

My plan was to install the RSA app on my rooted Nexus 5, provision a token and then analyze the application DB.
The application DB has a `SERIALNUMBER` and `ROOTSEED` columns. The serial number was written in plain text but the root seed was a huge binary blob.
Turns out we need some reversing after all ...

This is where I decided to give [Frida](https://frida.re/) a try. Frida is a dynamic instrumentation toolkit that supports all major platforms, including Android.
It allows you to install hooks into a target process that can trace or change execution flows. It's pretty amazing!

I managed to extract the serial number and the root seed from [RSA SecurID Authenticate](https://play.google.com/store/apps/details?id=com.rsa.via) ver.3.1.0 with the following script:

```javascript
function bytes2hex(array) {
    var result = '';
    for (var i = 0; i < array.length; ++i) {
        result += ('0' + (array[i] & 0xFF).toString(16)).slice(-2);
    }
    result += ' (' + array.length + ' bytes)'
    return result;
}

Java.perform(function() {
    Java.use('com.rsa.securidlib.g.gg').$init.overload('[B').implementation = function(byteArr) {
        if (byteArr.length == 16) {
            console.log("Seed: " + bytes2hex(byteArr));
        }
        return this.$init(byteArr);
    }
    Java.use('com.rsa.securidlib.android.AndroidSecurIDLib').getNextOtpWithDataProtection.implementation = function(str, byteArr, map) {
        console.log("Serial number: " + str);
        return this.getNextOtpWithDataProtection(str, byteArr, map);
    }
}); 
```
Once you have the serial number and the seed, you can use [this script](https://gist.github.com/dlenski/d6d4df40c8dd538339f750902d68bcfb)
to get the token into a CTF format (expiration doesn't matter):
```bash
$ SEED=<seed> SN=<serial_number> EXPIRATION=2030/12/31 ./make_RSA_token.sh
```
and then import it into `stoken`:
```bash
$ stoken import --token=<ctf_token>
```
Cheers :)
