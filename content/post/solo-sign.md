+++
title = "Signing files with Solo"
date = "2020-03-23T12:22:20+02:00"
tags = []
topics = []
description = ""
+++

[Solo](https://github.com/solokeys/solo) is FIDO2 security key with open hardware and firmware. I have been following the project and using the key for quite a while now.
It shares my believe that security solutions must be open. Solo is not only open but it also have developer edition, called
[Solo Hacker](https://solokeys.com/products/solo-hacker), which allows firmware modifications.

I've been dealing with digital signatures a lot lately, so I thought why not use my Solo key to sign files?
After all it can generate ECC keys and compute ECDSA signatures, so it should be straightforward. Well, not really.

First, let's see how keys are generated. I'll use the standard `fido2-cred` tool to create a new credential.
We put some arguments (which are not really important here) in the `cred_param` file and then run `fido2-cred`:

```bash
# Create a new credential and save the id and the public key of the credential in "cred"
$ echo credential challenge | openssl sha256 -binary | base64 > cred_param
$ echo relying party >> cred_param
$ echo user name >> cred_param
$ dd if=/dev/urandom bs=1 count=32 | base64 >> cred_param
$ fido2-cred -M -i cred_param /dev/hidraw6 | fido2-cred -V -o cred
$ cat cred
8BNpSl5onmiOYJNb68ZroWJtJvveCLXyC7l/gVFG0uF/kq6wOISXyMPTdcFX7nIGmKx4eL6HCtjxqpk3L6xdtFtUGgAAAA==
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE2T3KDkdhtOqVODP+4YH5k/yMzeAq
atKglLqflwzZTdHBvs6S1V9qsTC1l65DJCPZIn8MQYgsVvgwbalcgRp5Cw==
-----END PUBLIC KEY-----
```

Solo generates a new credential and returns the public key and the credential id (base64 encoded). You can think of the credential id
as an opaque handle which Solo derives the private key from. This method is called [key wrapping](https://docs.solokeys.io/solo/fido2-impl/#key-wrapping).

Let's say that we want to sign the file `file.dat` with this credential.
Since Solo uses ECDSA with SHA256, I would expect that executing `fido2-assert` and specifying `SHA256(file.dat)` as client data hash will produce the
file signature. But it doesn't work that way as we can see from the definition of the `GetAssertion` operation in the [WebAuthn spec](https://www.w3.org/TR/webauthn/#op-get-assertion):

{{% fluid_img class="pure-u-1-1" src="/images/fido-signature-formats-figure2.svg" caption="GetAssertion" %}}

The Solo authenticator doesn't sign the client data hash but the _concatenation_ of the authenticator data and the client data hash.
The former is a binary blob which contains flags, counter and other stuff.
It turns out there is no way to sign an arbitrary SHA256 hash using only the operations defined by the standard.
Of course, this is fine because the purpose of the WebAuthn standard is to define secure web authentication, not file signing.

Now let's look into another FIDO2 standard -- [CTAP2](https://fidoalliance.org/specs/fido-v2.0-rd-20170927/fido-client-to-authenticator-protocol-v2.0-rd-20170927.html) -- which describes the communication protocol between the authenticator and the platform. Section 6.1 defines the command values for each of the standard operations but also leaves room for vendor specific commands from `authenticatorVendorFirst(0x40)` to `authenticatorVendorLast(0xBF)`. This is great because we can add new commands with vendor specific functionality and assign them numbers from this range. I picked `0x50` and implemented a [new command](https://github.com/solokeys/solo/pull/397) for signing an arbitrary SHA256 hash. The modified [firmware](/files/firmware-3.1.3-2-gdd17854.hex) can be flashed to Solo Hacker like this:
```bash
$ solo program bootloader firmware-3.1.3-2-gdd17854.hex
```

The last step is [using](https://github.com/solokeys/solo-python/pull/67) the new vendor command in `solo-python` to implement file signing. The syntax is "`solo key sign-file <cred-id> <filename>`". It computes SHA256 hash of the specified file and invokes the vendor command, passing the credential id and the hash as arguments. Here is an example:
```bash
$ solo key sign-file 8BNpSl5onmiOYJNb68ZroWJtJvveCLXyC7l/gVFG0uF/kq6wOISXyMPTdcFX7nIGmKx4eL6HCtjxqpk3L6xdtFtUGgAAAA== file.dat
810743111f7f70cb64b5ef07b4930c2d2b298de743286609905b6f7873738a76  file.dat
Please press the button on your Solo key
Saving signature to file.dat.sig
```

The output is a DER encoded signature which is saved to `file.dat.sig`. We can verify the signature with `openssl`, using the public key from above:

```bash
$ cat public.pem
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE2T3KDkdhtOqVODP+4YH5k/yMzeAq
atKglLqflwzZTdHBvs6S1V9qsTC1l65DJCPZIn8MQYgsVvgwbalcgRp5Cw==
-----END PUBLIC KEY-----
$ openssl dgst -verify public.pem -sha256 -signature file.dat.sig file.dat
Verified OK
```

Finally, I will try to list some pros and cons of using this method for signing files.

 * Pros: I believe this is simpler alternative to let's say PGP. You also get the same protection from malware as with standard FIDO2 operations thanks to user presence test and PIN authentication.
 * Cons: One big disadvantage is that there is no way to backup your keys. You cannot import existing keys to the authenticator and there is no way to export keys from the authenticator.

For any questions or comments you can use the Github [issue](https://github.com/solokeys/solo/issues/395) that I have opened.

Cheers :)
