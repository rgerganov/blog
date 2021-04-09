+++
title = "SoftHSMv2 internals"
date = "2021-04-13T10:52:01+03:00"
tags = []
topics = []
description = ""
+++
[SoftHSMv2](https://github.com/opendnssec/SoftHSMv2) is a software implementation of the PCKS#11 interface.
It is often used as replacement for real HSM devices in test environments where protecting key material is not a strong requirement.
In this post I will explain how the state of SoftHSMv2 is persisted, the security behind it and what can be improved.


Tokens and objects
===
Token is the PKCS#11 term for something that stores cryptographic objects and performs cryptographic operations.
In SoftHSMv2 each token is organized as a directory containing files that represent token's objects.
All token directories have a common root which by default is `/var/lib/softhsm/tokens`.
Object files have an `.object` extension:

```sh
$ ls /var/lib/softhsm/tokens/643f45af-4f11-54c9-6edb-3578f9e3ea47/*object
/var/lib/softhsm/tokens/643f45af-4f11-54c9-6edb-3578f9e3ea47/7bfd8494-0a71-a984-cb8e-97dd6036dce8.object
/var/lib/softhsm/tokens/643f45af-4f11-54c9-6edb-3578f9e3ea47/7cb6bfc2-3526-18fe-1c02-f45a94e482c4.object
/var/lib/softhsm/tokens/643f45af-4f11-54c9-6edb-3578f9e3ea47/token.object
```

Object encryption
===
Each token is initialized with user PIN and SO PIN. SoftHSMv2 is using the user PIN to derive AES 256bit master key.
For every token it also generates a random AES token key which is used to encrypt and decrypt sensitive object attributes in the corresponding token.
Finally, SofthHSMv2 encrypts the token key with the master key and saves it to `token.object` in the token directory.
This is the pseudocode for all of this:

```
salt = RAND(8)
masterKey = KDF(salt, PIN)
tokenKey = RAND(32)
IV = RAND(16)
magic = "RJR"
encryptedBlob = AES256-CBC(magic || tokenKey, masterKey, IV)
save("token.object", salt || IV || encryptedBlob)
```
You can find the real C++ implementation in [SecureDataManager.cpp](https://github.com/opendnssec/SoftHSMv2/blob/develop/src/lib/data_mgr/SecureDataManager.cpp) and [RFC4880.cpp](https://github.com/opendnssec/SoftHSMv2/blob/develop/src/lib/data_mgr/RFC4880.cpp).

So what is the purpose of these magic bytes which are concatenated with the `tokenKey`? This is basically a poor man's implementation of authenticated encryption.
When a user tries to login with a PIN, SoftHSMv2 determines if the specified PIN is correct by decrypting the `encryptedBlob` and checking if it starts with the magic bytes.
I will talk about how this can be improved but first let's see an example for how to derive `masterKey` and `tokenKey` if we know the user PIN.

Example
---
Object files can be dumped with the `softhsm2-dump-file` utility. Let's dump `token.object` which contains the encrypted `tokenKey`:

```sh
$ softhsm2-dump-file /var/lib/softhsm/tokens/2d0d4809-45cc-93b1-4b3b-21ef36a33837/token.object
...
00 00 00 00 80 00 53 4d CKA_OS_USERPIN
00 00 00 00 00 00 00 03 byte string attribute
00 00 00 00 00 00 00 48 (length 72)
07 b1 79 16 02 74 f1 8f <..y..t..>
57 70 54 a9 81 ea 8e b8 <WpT.....>
28 a4 d4 23 d5 78 cc 16 <(..#.x..>
7f 7d a0 3d 25 54 a9 67 <.}.=%T.g>
5e ba 0e b2 90 8b ef 08 <^.......>
8b d8 44 21 8a 92 3d d8 <..D!..=.>
4a 83 6a 2c 68 70 d5 fe <J.j,hp..>
7b 46 bc 38 1b d0 e6 64 <{F.8...d>
35 49 8c 7c c2 e9 83 50 <5I.|...P>
```

The `CKA_OS_USERPIN` attribute is 72 bytes which is the concatenation of salt (8 bytes), IV (16 bytes) and encrypted blob (48 bytes).
Let's put these into shell variables:

```sh
salt=07b179160274f18f
IV=577054a981ea8eb828a4d423d578cc16
encryptedBlob=7f7da03d2554a9675eba0eb2908bef088bd844218a923dd84a836a2c6870d5fe7b46bc381bd0e66435498c7cc2e98350
```

With the correct PIN (1234) and [this](https://gist.github.com/rgerganov/fa58a7308c5d25f6652a0f25196d4181) script
we can obtain the `masterKey` and the `tokenKey` for this token:

```sh
$ ./softhsmv2.py 1234 $salt $IV $encryptedBlob
masterKey: 5fa8bdf96b1aa11da6aac165a0e4c3fd9d4b48838cbc58945e3ee27bc7fcf281
tokenKey: 302aa706005fb36002b65f65251d1621b183b04c680e3ec143210b7b63da9b85
```

Now we can use the `tokenKey` to decrypt the rest of the object files in this token.

Object integrity
===
SoftHSM is using AES-CBC and the `tokenKey` to encrypt all sensitive object attributes such as private keys.
The problem with this approach is that there is no way to guarantee the integrity of the object files.
If an attacker gets access to the filesystem, they can modify the object files leaving this undetected by the SoftHSM.
In this situation users will get incorrect results when using SoftHSM instead of error saying that the store has been tampered.

This problem can be easily solved by replacing AES-CBC with some of the authenticated modes of AES such as AES-GCM.
SoftHSM already supports AES-GCM and I have submitted this [PR](https://github.com/opendnssec/SoftHSMv2/pull/627).
However, backward compatibility with old tokens requires more work.

