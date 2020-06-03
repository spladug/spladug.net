+++
title = "Using GPG keys for a TLS CA"
description = "How to run a personal certificate authority using GPG keys stored on a smartcard (yubikey)"
date = 2020-05-24
+++

I wanted to have a personal Certificate Authority (<dfn id="CA">CA</dfn>) to
sign TLS certificates for my home network. Once the certificate for the the
CA was installed in my browsers, I'd be able to securely trust my local
services.

Since I have my GPG subkeys on a YubiKey, I wanted to re-use those private keys
for the CA root so that I didn't have to manage yet another master key. After
much fiddling, I figured out how to make [`gpgsm`][gpgsm] do what I wanted.

[gpgsm]: https://linux.die.net/man/1/gpgsm

# Background

Included in the [GPG] suite is a command called `gpgsm` which works on X.509
certificates, like those used in TLS. It is able to access the private keys in
your main GPG keyring&mdash;including ones that are actually on a smartcard
like a YubiKey&mdash;but keeps certificates and their associated public keys in
a separate database since the formats involved are quite different.

There are several steps to the certificate signing flow:

1. Generate a private key and certificate request (`gpgsm --generate-key`).
2. Sign the certificate (usually done by a third party CA, but we're the CA!)
3. Import the certificate into the public keyring (`gpgsm --import`).
4. Use/export the private key (`gpgsm --export-secret-key`).

In our case, we'll be combining steps 1 and 2 by generating the certificate and
signing it at the same time. However, even when doing this we still need to
explicitly import the certificate into the keyring (step 3) to be able to
access the generated private keys (step 4).

We'll be using `gpgsm` in `--batch` mode which reads a parameter file and
does the signing flow with minimal further input. The [parameter file format
is partially documented in GPG's manual][gpgsm-ua], but there are some
additional features that are [best found by reading its
source][gpgsm-source].

[GPG]: https://www.gnupg.org/
[gpgsm-ua]: https://www.gnupg.org/documentation/manuals/gnupg/CSR-and-certificate-creation.html
[gpgsm-source]: https://github.com/gpg/gnupg/blob/master/sm/certreqgen.c

# Extensions

While `gpgsm` knows a lot about the X.509 format, it doesn't have first-class
support for every kind of certificate extension we may need for our purposes.
Thankfully, it has an escape hatch in the form of the `Extension` parameter.
However, it requires us to encode the extensions manually.  For example, the
parameter `Extension: 2.5.29.19 c 30060101ff020100` encodes a [<q>basic
constraints</q>][basic-constraints] extension and can be understood as three
parts:

* the [OID] of the extension,
* whether the extension is **c**ritical or **n**ot (if critical, the client
  must fail validation if it doesn't understand the extension),
* and the [ASN.1] value of the extension [DER] encoded in hex.

The clearest way I found to generate these values is a tiny bit of Python
using the [`pyasn1`] and [`pyasn1_modules`] libraries, for example:

```python
from pyasn1.codec.der.encoder import encode
from pyasn1_modules.rfc5280 import BasicConstraints
from pyasn1_modules.rfc5280 import id_ce_basicConstraints

bc = BasicConstraints()
bc["cA"] = True
bc["pathLenConstraint"] = 0
print(id_ce_basicConstraints, "c", encode(bc).hex())
```

which prints out the `2.5.29.19 c 30060101ff020100` we saw above.

[basic-constraints]: https://www.alvestrand.no/objectid/2.5.29.19.html
[OID]: https://en.wikipedia.org/wiki/Object_identifier
[ASN.1]: https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One
[DER]: https://en.wikipedia.org/wiki/Distinguished_Encoding_Rules#DER_encoding
[`pyasn1`]: https://pypi.org/project/pyasn1/
[`pyasn1_modules`]: https://pypi.org/project/pyasn1-modules/

# Creating a CA certificate

The first thing we need to do is generate our root CA certificate. This
certificate is what we will install in the trust roots of browsers etc. to
allow secure access to the resources we sign with the CA's private key.

Despite the `--generate-key` command's name, we won't be generating a new
private key during this step. Instead, we tell it to use the signing key
(`[S]`) already in our keychain by specifying its <q>keygrip</q> which is
used to identify keys.  To get the keygrip, run the following:

```console
$ gpg --with-keygrip --list-keys
pub   rsa3072 2018-06-30 [C] [expires: 2021-05-02]
      5B5A70814529DDD6F45995A9A1986BFD48E8FD1E
      Keygrip = E804A37B63A55D6C86E852F1CB7B2173EDF2A398
uid           [ultimate] Neil Williams <neil@spladug.net>
uid           [ultimate] Neil Williams <neil@reddit.com>
sub   rsa4096 2019-05-02 [S] [expires: 2021-05-02]
      Keygrip = A2038400550F36E022A149FF6569993C517A591C
sub   rsa4096 2019-05-02 [A] [expires: 2021-05-02]
      Keygrip = 16C410601F49FFFCE22D369095B88440FAF5365B
sub   rsa4096 2019-05-02 [E] [expires: 2021-05-02]
      Keygrip = C38659A0687A0DF8458FF607B34F0A96EB4F5FC0
```

The signing key in this case has `A2038400`&hellip; as its keygrip. With the
keygrip determined, we can make a parameters file for our new self-signed CA
certificate:

```
# this must be the first line
Key-Type: RSA
# the keygrip of our extant private key
Key-Grip: A2038400550F36E022A149FF6569993C517A591C
# and sign the certificate immediately with the same key
Signing-Key: A2038400550F36E022A149FF6569993C517A591C
# using SHA-256
Hash-Algo: SHA256
# this is the name that will be shown for your certificate
Name-DN: CN=spladug home network
# creation and expiration
Creation-Date: 2020-01-01 00:00
Expire-Date: 2021-01-01 00:00
# should increase when you renew.
Serial: 1
# extensions!
Authority-Key-Id: A2038400550F36E022A149FF6569993C517A591C
Subject-Key-Id: A2038400550F36E022A149FF6569993C517A591C
Extension: 2.5.29.19 c 30060101ff020100
```

Put that into a file named `ca.params` and then generate the certificate with
`gpgsm`:

```console
$ gpgsm --armor --generate-key --batch ca.params > ca.crt
gpgsm: about to sign the certificate for key:
&A2038400550F36E022A149FF6569993C517A591C
gpgsm: certificate created
```

The output should be a valid CA root certificate that you can import into
browsers and other trust roots. You can inspect it with `openssl`:

```console
$ openssl x509 -text -in ca.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = spladug home network
        Validity
            Not Before: Jan  1 00:00:00 2020 GMT
            Not After : Jan  1 00:00:00 2021 GMT
        Subject: CN = spladug home network
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
                Modulus:
                    ... snip ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Subject Key Identifier:
                A2:03:84:00:55:0F:36:E0:22:A1:49:FF:65:69:99:3C:51:7A:59:1C
            X509v3 Authority Key Identifier:
                keyid:A2:03:84:00:55:0F:36:E0:22:A1:49:FF:65:69:99:3C:51:...

... snip ...
```

Since we don't need to export the private key for the CA to send it elsewhere
(and _can't_ if it's on a smartcard) we're done here.

# Making endpoint certificates

For each endpoint certificate, we'll go through roughly the same flow except
that we'll be generating a new private key each time and instead of
self-signing we'll sign with the CA's private key.

To start out, we'll make a new parameter file for the endpoint certificate:

```
# again, this must be the first line
Key-Type: RSA
# instead of Key-Grip we specify how large we want a newly generated
# private key to be
Key-Length: 2048
# then sign the certificate immediately with the CA key
Signing-Key: A2038400550F36E022A149FF6569993C517A591C
# using SHA-256
Hash-Algo: SHA256
# leave the creation date as "now" and set an expiration
Expire-Date: 2021-01-01 00:00
# set the serial to a random value
Serial: random
# what purposes are we certifying this key good for
Key-Usage: sign, encrypt
# identify the CA certificate
Issuer-DN: CN=spladug home network
Authority-Key-Id: A2038400550F36E022A149FF6569993C517A591C
# the name gets more important for endpoint certs
Name-DN: CN=example.com
# Name-DNS translates to subject alternative names
Name-DNS: foo.example.com
Name-DNS: bar.example.com
# this is the basic constraints extension with CA=false
Extension: 2.5.29.19 n 3000
```

Just like before, put this in a file like `endpoint.params` and use it to
generate a key and signed certificate:

```
$ gpgsm --armor --generate-key --batch endpoint.params > endpoint.crt
gpgsm: about to sign the certificate for key:
&A2038400550F36E022A149FF6569993C517A591C
gpgsm: certificate created
```

You will be prompted for a passphrase for protecting (encrypting) the private
key. You can leave it blank to leave the key unencrypted if desired.

Like before, you can inspect the certificate with `openssl`:

```console
$ openssl x509 -text -in endpoint.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4291214621359795366 (0x3b8d74f6591ceca6)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = spladug home network
        Validity
            Not Before: May 29 05:54:37 2020 GMT
            Not After : Jan  1 00:00:00 2021 GMT
        Subject: CN = example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    ... snip ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:foo.example.com, DNS:bar.example.com
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Authority Key Identifier:
                keyid:A2:03:84:00:55:0F:36:E0:22:A1:49:FF:65:69:99:3C:51:...

            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation, Key Encipherment, Data
                Encipherment

... snip ...
```

This time, we're not done. We need to extract the private key to use in the
endpoint. First off, import the newly generated certificate:

```console
$ gpgsm --import endpoint.crt
gpgsm: total number processed: 1
gpgsm:               imported: 1
```

Then get the ID of the key (_not_ keygrip, but we'll use that later):

```console
$ gpgsm --list-keys --with-keygrip
/home/spladug/.gnupg/pubring.kbx
--------------------------------------------
           ID: 0x2FD3341C
          S/N: 01
       Issuer: /CN=spladug home network
      Subject: /CN=spladug home network
     validity: 2020-01-01 00:00:00 through 2030-01-01 00:00:00
     key type: 4096 bit RSA
 chain length: 0
  fingerprint: E1:D0:9E:5E:D0:88:A3:78:AF:49:9D:9D:B0:F9:73:43:2F:D3:34:1C
      keygrip: A2038400550F36E022A149FF6569993C517A591C

           ID: 0xA0CBFE60
          S/N: 3B8D74F6591CECA6
       Issuer: /CN=spladug home network
      Subject: /CN=example.com
          aka: (dns-name foo.example.com)
          aka: (dns-name bar.example.com)
     validity: 2020-05-29 05:54:37 through 2021-01-01 00:00:00
     key type: 2048 bit RSA
    key usage: digitalSignature nonRepudiation keyEncipherment dataEncipherment
  fingerprint: 23:11:53:73:83:48:0A:00:4C:C8:E1:A9:6D:08:26:82:A0:CB:FE:60
      keygrip: 03AC0260D114811D59215DE7126D448E510C1186
```

In this example, there are two keys: the CA and the new endpoint certificate.
If you have more endpoints or are doing other things with `gpgsm` there will
be more. The key we just made is `0xA0CBFE60` so that's the value we care
about.

With the key ID figured out, we can export the private key:

```
$ gpgsm --armor --export-secret-key-raw 0xA0CBFE60 > endpoint.key
```

We now have a certificate (`endpoint.crt`) and private key (`endpoint.key`)
that can be installed in our endpoint.

As a final step, clean up the private key from our local keychain so the key
only exists on the endpoint going forward. To do this, manually delete the
private key file from the GPG private key directory using the keygrip listed
above (`03AC0260D114811D59...` in this case).

```console
$ rm $GNUPGHOME/private-keys-v1.d/03AC0260D114811D59215DE7126D448E510C1186.key
```

And we're done!

# Other reading and prior art

The following, in addition to things linked above,  were both extremely useful
to me in figuring out how all of this worked.

* <https://www.gnupg.org/documentation/manuals/gnupg/Howto-Create-a-Server-Cert.html>
* <https://github.com/jymigeon/gpgsm-as-ca/tree/master/card-signing>
