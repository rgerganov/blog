+++
title = "Making a Chromecast receiver"
date = "2023-09-27T14:28:02+03:00"
tags = []
topics = []
description = "Project Shanocast"
+++

I made a Chromecast receiver which works on Linux. It is called [Shanocast](https://github.com/rgerganov/shanocast) and it can mirror a Chrome tab or the entire desktop. Here is a demo:

{{< rawhtml >}} 

<video width=100% controls>
    <source src="/videos/shanocast-demo.mp4" type="video/mp4">
    Your browser does not support the video tag.  
</video>

{{< /rawhtml >}}

The implementation is based on [Openscreen](https://chromium.googlesource.com/openscreen/) which is an open-source implementation of the Chromecast protocol.
The tricky part is the receiver authentication. Google Chrome authenticates the receiver and refuses to stream to it if the authentication fails. I will explain how I solved this problem in this post.

## Receiver authentication

Clients are using challenge-response protocol to authenticate the Chromecast receiver. These are the relevant messages from the [protocol](https://chromium.googlesource.com/openscreen/+/refs/heads/main/cast/common/channel/proto/cast_channel.proto):
```
// Messages for authentication protocol between a sender and a receiver.
message AuthChallenge {
  optional SignatureAlgorithm signature_algorithm = 1
      [default = RSASSA_PKCS1v15];
  optional bytes sender_nonce = 2;
  optional HashAlgorithm hash_algorithm = 3 [default = SHA1];
}

message AuthResponse {
  required bytes signature = 1;
  required bytes client_auth_certificate = 2;
  repeated bytes intermediate_certificate = 3;
  optional SignatureAlgorithm signature_algorithm = 4
      [default = RSASSA_PKCS1v15];
  optional bytes sender_nonce = 5;
  optional HashAlgorithm hash_algorithm = 6 [default = SHA1];
  optional bytes crl = 7;
}
```
It works like this:

1. The receiver generates self-signed `peer_certificate` and starts a TLS server with it
2. The client (Google Chrome) sends an `AuthChallenge` message to the receiver
3. The receiver signs `(sender_nonce || peer_certificate)` with its `client_auth_certificate` and sends the signature back to the client in an `AuthResponse` message
4. The client verifies that `client_auth_certificate` is signed by a trusted CA and that the signature in the `AuthResponse` message is valid

It looks like the only way to implement this authentication scheme on the receiver side is to have a `client_auth_certificate` signed by Google and its corresponding private key. 

Turns out there is also another way.

## AirReceiver app

I found an Android app called [AirReceiver](https://play.google.com/store/apps/details?id=com.softmedia.receiver.lite&hl=en&gl=US) which can act as a Chromecast receiver. I was able to stream my desktop Chrome browser to my phone with this app! How do they do it?
I really don't think that Google will issue a Chromecast certificate to some random app developer. Maybe they are using a certificate from a rooted Chromecast device or something like this? Let's take a look at the `peer_certificate`:

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 85760713 (0x51c9ac9)
        Signature Algorithm: sha1WithRSAEncryption
        Issuer: CN = 4aa9ca2e-c340-11ea-8000-18ba395587df
        Validity
            Not Before: Sep 26 00:00:00 2023 GMT
            Not After : Sep 28 00:00:00 2023 GMT
        Subject: CN = 4aa9ca2e-c340-11ea-8000-18ba395587df       
```

It is valid only for 48 hours. After playing with this for a while, I had the following observations:

1. The `peer_certificate` is always valid for 48 hours and the app generates a new one only after the previous one expires.
2. The `peer_certificate` is always using the same RSA key pair.
3. The `sender_nonce` is missing in the `AuthResponse` message and the returned signature is only for the `peer_certificate`. This means that the returned signature is always the same for the entire lifetime of the `peer_certificate` (48 hours).

Apparently Google Chrome is totally fine with such authentication responses and doesn't mind if `sender_nonce` is missing in the response. I looked at how `AuthResponse` is verified in Openscreen and I found a boolean flag called [enforce_nonce_checking](https://chromium.googlesource.com/openscreen/+/refs/heads/main/cast/sender/channel/cast_auth_util.cc#210). If this flag is set to `false`, the sender doesn't check the received nonce and verifies the signature without it. Google Chrome is using Openscreen for its cast implementation and for some reason `enforce_nonce_checking` is set to `false`.

At this point things are pretty much clear. Without the nonce check, the whole authentication is vulnerable to replay attacks and the AirReceiver app is doing exactly this. It has precomputed signatures from somewhere (probably a rooted Chromecast) for each `peer_certificate` that it generates. The signature is 256 bytes, it changes every 2 days, so we need ~45KB of storage to store all the signatures for a year.

So we need to take two things from AirReceiver to make our own Chromecast receiver:
1. The private key for the `peer_certificate`
2. The precomputed signatures

`client_auth_certificate` and `intermediate_certificate` are already public.

## Reversing AirReceiver

Needless to say, AirReciever is heavily obfuscated. A significant part of its implementation is in the native `libAirReceiver.so` library which has OpenSSL statically linked. I found an excellent tool called [jnitrace](https://github.com/chame1eon/jnitrace) which is based on [frida](https://frida.re/) and can be used to trace JNI calls. Tracing the JNI calls upon start revealed the `peer_certificate` and its private key:
```
$ jnitrace -l libAirReceiver.so com.softmedia.receiver.lite

...
           /* TID 10690 */
   2302 ms [+] JNIEnv->ReleaseStringUTFChars
   2302 ms |- JNIEnv*          : 0x7775f8c180
   2302 ms |- jstring          : 0x7775f9bc00
   2302 ms |- char*            : 0x7775f9bc00
   2302 ms |:     {"cpu":"-----BEGIN CERTIFICATE-----\nMIIDqzCCApOgAwIBAgIEUl20yDANBgkqhkiG9w0BAQUFADB9MQswCQYDVQQGEwJV\nUzETMBEGA1UECAwKQ2FsaWZvcm5pYTEWMBQGA1UEBwwNTW91bnRhaW4gVmlldzET\nMBEGA1UECgwKR29vZ2xlIEluYzESMBAGA1UECwwJR29vZ2xlIFRWMRgwFgYDVQQD\nDA9FdXJla2EgR2VuMSBJQ0EwHhcNMTMxMDE1MjEzNDAwWhcNMzMxMDEwMjEzNDAw\nWjCBgDETMBEGA1UEChMKR29vZ2xlIEluYzETMBEGA1UECBMKQ2FsaWZvcm5pYTEL\nMAkGA1UEBhMCVVMxFjAUBgNVBAcTDU1vdW50YWluIFZpZXcxEjAQBgNVBAsTCUdv\nb2dsZSBUVjEbMBkGA1UEAxMSV0RGVDMgRkE4RkNBODk1RDU5MIIBIjANBgkqhkiG\n9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyuJeiL3ku+mTK2EwOdbqf0APEeqqa0HOTA0V\nCXQXGOJdEXboCnNkSb46A0Cn1OO7/2R7Ex0OlhSoYv/2xKgbOpLSp3gqXl1aRNMN\nD1i+YVdDzdS6F9IWvx1iTYSuPsxUSH5ECzEl5lvDSWXPIn54AmkVG04xuVETfLLQ\ny8dbZsjh89ODPTDRLDODf78lfKagvHmFnWPXBdPPoUYokmLH3Iuxp7zl9G7oxL/+\nP6VPkgylAzGqnSvmtMdDs6Lz/GZ2KX5WMKB8n+c2g1UMjPgaiut67wf/V6ltvQXN\n95uue9JiHBECgZWbYTadI9aIpdbExwbCdpeRFs8vsx5ITmFlLwIDAQABoy8wLTAJ\nBgNVHRMEAjAAMAsGA1UdDwQEAwIHgDATBgNVHSUEDDAKBggrBgEFBQcDAjANBgkq\nhkiG9w0BAQUFAAOCAQEAc/T1hQ01kjkETg2lLXPIcYG3nP5RXIyDwnXlNWsHVzZl\nz/Vvqq/rLmQwJjdQjVWjP+mZlw6Y3O8q0cVKUEWVtk4GGk6WHfCM+s/jeznaeEGg\n3LI2TuUCyD2RkbaQozSQGjvU1NXyI/fYNBociBfkf594pnRS/sXOUisuo8IyuwN/\no3CeiX+FAkizYiXhrUYCvPQpFtOgHQbSuNeDE2R/HKyOKkW/DlDRWO9tQa+O9SLi\n/UqCsaAxOqlOg32PW1rt1fR5CgTT5A3kfExXoA4n0LJ+CEH8UenddEuh5KZ+xuUP\nWkxPQTOEAE0MscxdtvrtOxb9ZpTfUahdnTeu2E4PkQ==\n-----END CERTIFICATE-----\n","ica":"-----BEGIN CERTIFICATE-----\nMIIDhzCCAm+gAwIBAgIBATANBgkqhkiG9w0BAQUFADB8MQswCQYDVQQGEwJVUzET\nMBEGA1UECAwKQ2FsaWZvcm5pYTEWMBQGA1UEBwwNTW91bnRhaW4gVmlldzETMBEG\nA1UECgwKR29vZ2xlIEluYzESMBAGA1UECwwJR29vZ2xlIFRWMRcwFQYDVQQDDA5F\ndXJla2EgUm9vdCBDQTAeFw0xMjEyMTkwMDQ3MTJaFw0zMjEyMTQwMDQ3MTJaMH0x\nCzAJBgNVBAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYwFAYDVQQHDA1Nb3Vu\ndGFpbiBWaWV3MRMwEQYDVQQKDApHb29nbGUgSW5jMRIwEAYDVQQLDAlHb29nbGUg\nVFYxGDAWBgNVBAMMD0V1cmVrYSBHZW4xIElDQTCCASIwDQYJKoZIhvcNAQEBBQAD\nggEPADCCAQoCggEBALwigL2A9johADuudl41fz3DZFxVlIY0LwWHKM33aYwXs1Cn\nuIL638dDLdZ+q6BvtxNygKRHFcEgmVDN7BRiCVukmM3SQbY2Tv/oLjIwSoGoQqNs\nmzNuyrL1U2bgJ1OGGoUepzk/SneO+1RmZvtYVMBeOcf1UAYL4IrUzuFqVR+LFwDm\naaMn5gglaTwSnY0FLNYuojHetFJQ1iBJ3nGg+a0gQBLx3SXr1ea4NvTWj3/KQ9zX\nEFvmP1GKhbPz//YDLcsjT5ytGOeTBYysUpr3TOmZer5ufk0K48YcqZP6OqWRXRy9\nZuvMYNyGdMrP+JIcmH1X+mFHnquAt+RIgCqSxRsCAwEAAaMTMBEwDwYDVR0TBAgw\nBgEB/wIBATANBgkqhkiG9w0BAQUFAAOCAQEAi9Shsc9dzXtsSEpBH1MvGC0yRf+e\nq9NzPh8i1+r6AeZzAw8rxiW7pe7F9UXLJBIqrcJdBfR69cKbEBZa0QpzxRY5oBDK\n0WiFnvueJoOOWPN3oE7l25e+LQBf9ZTbsZ1la/3w0QRR38ySppktcfVN1SP+Mxyp\ntKvFvxq40YDvicniH5xMSDui+gIK3IQBiocC+1nup0wEfXSZh2olRK0WquxONRt8\ne4TJsT/hgnDlDefZbfqVtsXkHugRm9iy86T9E/ODT/cHFCC7IqWmj9a126l0eOKT\nDeUjLwUX4LKXZzRND5x2Q3umIUpWBfYqfPJ/EpSCJikH8AtsbHkUsHTVbA==\n-----END CERTIFICATE-----\n","pr":"-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAwmw+d820+BDW0zQI1T4Yot2vrANCteILcDUjZN72TLZGRH8r\nqUapcQlPQqUXrK/nJjeHx9gz8w1xZXqT7ClpAMgKAwyd3iLaqd1JYb4rzPsNGvXI\nKl5B21aCVi05hIc8Bo7Nq2l8rAmaTw1G43K355SNe7a+ZGU9CujOjAYYtvhZ+uZ0\n6X/h44HeJh/YqTSSBXcMriiinvEtXVKP/cJrbf3oaC/0ZJWCu4kuLsomJErX+gPP\nDVLWI0ai5J+GlrwyzUTEYyrH+z/gKFLRulQKhecJXQw2k6bqzm9lCDYTSwtlyc7u\nFS6D7k8W28goaNs54UeWU1d7AgXx3s90+UxBzwIDAQABAoIBAEFfTBHUZQkUAGe7\nk0zAOGBq0eqwnfmyK85qz5/XKFHa5/2YFQIx9D9BthjekftKmhpLiag0liMfXgWV\nFa/OrLPKjzM/RsWuSn/bHBV1cBzYPSvXgJpeXx51FBYN1s0s+43o7la4fWcLQ4tZ\nF4DazeNcG8aBR7tSHxhP90M1uZGrkUz9k2qxP5rrlF2peaKKaRqUsdPvlFWGY7+4\nb0nfYhj+gOVTsTDokEhFvrO438GEG08QR5AweQ0tqVzm7KTUW5Ihgn+rb2wB0GoR\nSl4nshw6dkZfIH1N8TNywYTV0l9WVfXPFS3cZxq1G/mqRD4m1G/891E/kr6OPyUB\nf0DQleECgYEA9H8jjK6R/GRPcL4MKij0/ghNYt5KCXcOJfGt+gTq7V0DO20sE3ct\nX+1/sXvGHU+wgSsrqmbGwRm9KfRZIWFBgjW5JbHLpvnMgwV3qVCGeH0lOxcuGyYx\nEjy4qeJYiS1j5s1AimzxklJga85afvOu+JgEruaxlySw0kURdyHmv/0CgYEAy5H8\nng/YKycN0VkLvljLmDTEB6xB8l0/oLLU+NUHfuX87kWjYP12/gmuI+ESZkyWqI6s\nwbY0++yxx0GX1pjdnljYeRXcyvNnC2XYXVkwgDDaf5csPEbFADEC7f19upHpm2Cv\niKLIYyTr8RiZ+LrLecKfho5xtHzN1MshtkBrVLsCgYAfZL/MzZGDJeIpaM2pEC88\n+xXsrvw0sOvJJXogU0dTCRFkLQVuzmuuGJG/2VO76cKRI1js/Vth6gsm+vAC4DkI\nHhvS4jxzCTogTLBrtiI+EFuadcR+ye2dGNzhO2YA3yontY0m+QwfrKIi1ZE7IdEC\nrIpVZtvAu35U0XeHo3u8hQKBgCAeNGEr1stYKhHxnqy1jcnB6XvcbbszgypzjK6F\nzdzzpGhjjFdtJi0GkfcPN7v0MYD+obseaFWnDpWFf9NX4v9svRq9nExZAtUFiJGR\n1Nkk3BRtYYlRERvqn6+04vVguB7PrmI8bKlX1fIAE6rurdPUJR8xsjbryf3c3sDG\ngSipAoGAMl65bMTHhNEncoa9+n9CW7rQBQc0uzwG3Q/wvoG23j/+lp8IrvIqzVrv\no8fmaGymUsT9siq/mjTe60AmiFwoYiXVYE1/V58oNQPg11klAACs9MT1qTa5P//X\nEQqAdblKGF2/RDqaDAxYUIXwU/VJ2CZxLX9nOQm9DwUljfY4+rQ=\n-----END RSA PRIVATE KEY-----\n","pu":"-----BEGIN CERTIFICATE-----\nMIIC2jCCAcKgAwIBAgIEBRyayTANBgkqhkiG9w0BAQUFADAvMS0wKwYDVQQDDCQ0\nYWE5Y2EyZS1jMzQwLTExZWEtODAwMC0xOGJhMzk1NTg3ZGYwHhcNMjMwOTI2MDAw\nMDAwWhcNMjMwOTI4MDAwMDAwWjAvMS0wKwYDVQQDDCQ0YWE5Y2EyZS1jMzQwLTEx\nZWEtODAwMC0xOGJhMzk1NTg3ZGYwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK\nAoIBAQDCbD53zbT4ENbTNAjVPhii3a+sA0K14gtwNSNk3vZMtkZEfyupRqlxCU9C\npResr+cmN4fH2DPzDXFlepPsKWkAyAoDDJ3eItqp3UlhvivM+w0a9cgqXkHbVoJW\nLTmEhzwGjs2raXysCZpPDUbjcrfnlI17tr5kZT0K6M6MBhi2+Fn65nTpf+Hjgd4m\nH9ipNJIFdwyuKKKe8S1dUo/9wmtt/ehoL/RklYK7iS4uyiYkStf6A88NUtYjRqLk\nn4aWvDLNRMRjKsf7P+AoUtG6VAqF5wldDDaTpurOb2UINhNLC2XJzu4VLoPuTxbb\nyCho2znhR5ZTV3sCBfHez3T5TEHPAgMBAAEwDQYJKoZIhvcNAQEFBQADggEBAExY\nK77zCdl6Xg8JnBL6bX90hbhoBzns0phEFxE1LqPnmCCYYIXyOmPg+YSieNTvYbVb\nuBziNLfqeW9+DvDSBcl1vWs0+oQM6O4YzEsx14BBRYo/fpccK6gs3/iPdaPYZJ6P\nm8kC/N0e+xQfF3hZJVE9RQ79RnpF0FJO7hE/8Dc3S0HJQBVvZtqC65VTocWP8HPl\nqLstNAxZOJvYiluUXNzoTbnpkhhMZa4hcs275sNoQ+nzhhlJtz4DevBNMaoHd23U\njIALUDGsIxF1xUNkSPbrfNWGUxerg+Yxr/GTqAJmNot+AGsccCzxINZNyrHv8/v6\n7zBHGyBa6B45hvxVGPc=\n-----END CERTIFICATE-----\n","sha256_sig":"kx5CjJk9WMXvtZeXnn0N/dsUaIv65skrQoSWfR85ucmejLRGkklCxvjs7rMQ36Yq5O2khfaIWKVxeedMKD/hS7mgui8NVBHL49k/MLPD3hMMSY19TysfReA4KVNX7WzajCad5zwyjpw/+5SyvbWmv0XsPx6uY4ymQrOAxQHSZBgOUxuJKf3aFbegSFEmrIWdu0WGMzsYUAJvXT9xQxN19Syknu1aKXZvHuTjvKM2oAQlkUOVaNzzNRhTELYVSJWX5B4z3n/NkM3hciFbo3CF4es2e08LqAS3h7r168+sbNxEX+7AsDFcr4Gb1IHX5DHpcmHaUgAyU80gNnh9r+vh8A==","sig":"cjBvXVL+LGPbUCP4j+vgLoUsL2fjctjiEBGfRnpMG8VsA9HktesreTTNIqbCpXqX5KCBvndAagX3X86op8tkDXrwyJn8iMxOdrWuoaPnuLYeSj9r9Cc2HJXTGO2mqwy94rWgzYodb8s9trr4bOk5i86z+cVxjt7Ai6huGJ6ru1rGenKCRQkV4MwVFi7IAz7fL2Eml1ztrOpe3Uo9B+wGz506iymM7wOL+3JLlbCl7lTcgPZn4CwYXYJi2fVj7m/lqZYiewnBQezGdqKAiBHNjIWftyDYfaTts06QbwfbkwGa9HjzF8plLAx2x9iXCNQYdmxQIM/ORd0J/JaGb1Pkbg=="}

   2302 ms -------------------------------------Backtrace-------------------------------------
   2302 ms |->       0x778fffc8ac: libAirReceiver.so!0x6d8ac (libAirReceiver.so:0x778ff8f000)
   2302 ms |->       0x778fffc8ac: libAirReceiver.so!0x6d8ac (libAirReceiver.so:0x778ff8f000)
...
```

I had no luck finding the precomputed signatures though. Fortunately, there is also another way for that -- if we change the current date on the phone, AirReceiver will regenerate the `peer_certificate` and then will start returning `AuthResponse` with the new signature. We can automate that easily on a rooted phone with `adb` and a simple binary which sends authentication requests to AirReceiver:

{{< gist rgerganov 856526a98c6f46c03f494fd89201ac1e >}}

In 15 minutes I have collected 795 signatures spanning from Aug-15-2023 to 21-Dec-2027.

## Putting it all together

Openscreen comes with a [standalone receiver](https://chromium.googlesource.com/openscreen/+/refs/heads/main/cast/standalone_receiver/) which implements the Chromecast protocol without any "official" certificates and keys.
I hacked this receiver to generate the same `peer_certificate` as AirReceiver with the same RSA key and return the same precomputed signatures. You can find the complete patch [here](https://github.com/rgerganov/shanocast/blob/master/shanocast.patch).
I have tested only on Linux but I guess it should work on other platforms as well.

I don't think this hack will work for long, so enjoy it while it lasts :)
