Title: Converting OpenSSH public keys
Date: 2011-5-8
Tags: openssh,rsa,cryptography,openssl,fixed

> I've posted a
> [followup](/posts/signing-data-with-ssh-agent)
> to this article that discusses ssh-agent.

For reasons best left to another post, I wanted to convert an SSH public
key into a PKCS\#1 PEM-encoded public key. That is, I wanted to go from
this:

    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCxLAe9fKl+HLULpQNasL0VBctX23AWK4FIsJ+x9kwPwixubvLRDU2Ah1WnJsfSth5PQqZYk8/QjjVmF92kcfCkNjE9ACa4I0g8CX2NZ3kZ2UdyWrSLYJxVSzou4HgSdH4B3p+/unsBRefs2W/2zLuAZt3hOUJSokmBE1NkrEugL4bUatwLDPHWQ1nH5RdANz+6PNClzTKVytGpDHqXG0oj9hkjur4bd94xZDfuFyhrtUrGiS7TaRVD29zwEPDnDfOVI9/qVQ+Xu3kivCPno1MwsQRYxnL+PnAQRa0kcfHAkgQOGIV599wMReNc81IgDkwL3WIL4m608MG+d4tJ+4j1 lars@madhatter.int.seas.harvard.edu


To this:

    -----BEGIN RSA PUBLIC KEY-----
    MIIBCgKCAQEAsSwHvXypfhy1C6UDWrC9FQXLV9twFiuBSLCfsfZMD8Isbm7y0Q1N
    gIdVpybH0rYeT0KmWJPP0I41ZhfdpHHwpDYxPQAmuCNIPAl9jWd5GdlHclq0i2Cc
    VUs6LuB4EnR+Ad6fv7p7AUXn7Nlv9sy7gGbd4TlCUqJJgRNTZKxLoC+G1GrcCwzx
    1kNZx+UXQDc/ujzQpc0ylcrRqQx6lxtKI/YZI7q+G3feMWQ37hcoa7VKxoku02kV
    Q9vc8BDw5w3zlSPf6lUPl7t5Irwj56NTMLEEWMZy/j5wEEWtJHHxwJIEDhiFeffc
    DEXjXPNSIA5MC91iC+JutPDBvneLSfuI9QIDAQAB
    -----END RSA PUBLIC KEY-----

If you have a recent version of OpenSSH (where recent means 5.6 or
later), you can just do this:

    ssh-keygen -f key.pub -e -m pem

If you don't have that, read on.

OpenSSH Public Key Format
=========================

The OpenSSH public key format is fully documented in [RFC
4253](http://tools.ietf.org/html/rfc4253#section-6.6). Briefly, an
OpenSSH public key consists of three fields:

-   The key type
-   A chunk of PEM-encoded data
-   A comment

What, you may ask, is PEM encoding? [Privacy Enhanced
Mail](http://en.wikipedia.org/wiki/Base64#Privacy-enhanced_mail) (PEM)
is a specific type of [Base64](http://en.wikipedia.org/wiki/Base64)
encoding...which is to say it is a way of representing binary data using
only printable ASCII characters.

For an `ssh-rsa` key, the PEM-encoded data is a series of (length, data)
pairs. The length is encoded as four octets (in big-endian order). The
values encoded are:

1.  algorithm name (one of (`ssh-rsa`, `ssh-dsa`)). This duplicates the
    key type in the first field of the public key.
2.  RSA exponent
3.  RSA modulus

For more information on how RSA works and what the exponent and modulus
are used for, read the Wikipedia article on
[RSA](http://en.wikipedia.org/wiki/RSA).

We can read this in with the following Python code:

    import sys
    import base64
    import struct

     # get the second field from the public key file.
     keydata = base64.b64decode(
       open('key.pub').read().split(None)[1])

     parts = []
     while keydata:
         # read the length of the data
         dlen = struct.unpack('>I', keydata[:4])[0]

         # read in <length> bytes
         data, keydata = keydata[4:dlen+4], keydata[4+dlen:]

         parts.append(data)

This leaves us with an array that, for an RSA key, will look like:

    ['ssh-rsa', '...bytes in exponent...', '...bytes in modulus...']

We need to convert the character buffers currently holding *e* (the
exponent) and *n* (the modulus) into numeric types. There may be better
ways to do this, but this works:

    e_val = eval('0x' + ''.join(['%02X' % struct.unpack('B', x)[0] for x in
         parts[1]]))
    n_val = eval('0x' + ''.join(['%02X' % struct.unpack('B', x)[0] for x in
         parts[2]]))

We now have the RSA public key. The next step is to produce the
appropriate output format.

PKCS\#1 Public Key Format
=========================

Our target format is a PEM-encoded PKCS\#1 public key.

PKCS\#1 is "the first of a family of standards called Public-Key
Cryptography Standards (PKCS), published by RSA Laboratories."
([Wikipedia](http://en.wikipedia.org/wiki/PKCS1)). You can identify a
PKCS\#1 PEM-encoded public key by the markers used to delimit the base64
encoded data:

    -----BEGIN RSA PUBLIC KEY-----
    ...
    -----END RSA PUBLIC KEY-----

This is different from an x.509 public key, which looks like this:

    -----BEGIN PUBLIC KEY-----
    ...
    -----END PUBLIC KEY-----

The x.509 format may be used to store keys generated using algorithms
other than RSA.

The data in a PKCS\#1 key is encoded using DER, which is a set of rules
for serializing ASN.1 data. For more information see:

-   The WikiPedia entry for [Distinguished Encoding
    Rules](http://en.wikipedia.org/wiki/Distinguished_Encoding_Rules)
-   [A Layman's Guide to a Subset of ASN.1, BER, and
    DER](http://luca.ntop.org/Teaching/Appunti/asn1.html)

Basically, ASN.1 is a standard for describing abstract data types, and
DER is a set of rules for transforming an ASN.1 data type into a series
of octets.

According to the [ASN module for
PKCS\#1](ftp://ftp.rsasecurity.com/pub/pkcs/pkcs-1/pkcs-1v2-1.asn), a
PKCS\#1 public key looks like this:

    RSAPublicKey ::= SEQUENCE {
       modulus           INTEGER,  -- n
       publicExponent    INTEGER   -- e
    }

We can generate this from our key data using Python's
[PyASN1](http://pyasn1.sourceforge.net/) module:

    from pyasn1.type import univ

    pkcs1_seq = univ.Sequence()
    pkcs1_seq.setComponentByPosition(0, univ.Integer(n_val))
    pkcs1_seq.setComponentByPosition(1, univ.Integer(e_val))

Generating the output
=====================

Now that we have the DER encoded key, generating the output is easy:

    from pyasn1.codec.der import encoder as der_encoder

    print '-----BEGIN RSA PUBLIC KEY-----'
    print base64.encodestring(der_encoder.encode(pkcs1_seq))
    print '-----END RSA PUBLIC KEY-----'

Appendix: OpenSSH private key format
====================================

Whereas the OpenSSH public key format is effectively "proprietary" (that
is, the format is used only by OpenSSH), the private key is already
stored as a PKCS\#1 private key. This means that the private key can be
manipulated using the OpenSSL command line tools.

The clever folks among you may be wondering if, assuming we have the
private key available, we could have skipped this whole exercise and
simply extracted the public key in the correct format using the
`openssl`command. We can come very close...the following demonstrates
how to extract the public key from the private key using `openssl`:

    $ openssl rsa -in sample.key -pubout
    writing RSA key
    -----BEGIN PUBLIC KEY-----
    MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsSwHvXypfhy1C6UDWrC9
    FQXLV9twFiuBSLCfsfZMD8Isbm7y0Q1NgIdVpybH0rYeT0KmWJPP0I41ZhfdpHHw
    pDYxPQAmuCNIPAl9jWd5GdlHclq0i2CcVUs6LuB4EnR+Ad6fv7p7AUXn7Nlv9sy7
    gGbd4TlCUqJJgRNTZKxLoC+G1GrcCwzx1kNZx+UXQDc/ujzQpc0ylcrRqQx6lxtK
    I/YZI7q+G3feMWQ37hcoa7VKxoku02kVQ9vc8BDw5w3zlSPf6lUPl7t5Irwj56NT
    MLEEWMZy/j5wEEWtJHHxwJIEDhiFeffcDEXjXPNSIA5MC91iC+JutPDBvneLSfuI
    9QIDAQAB
    -----END PUBLIC KEY-----

So close! But this is in x.509 format, and even though the OpenSSL
library supports PKCS\#1 encoding, there is no mechanism to make the
command line tools cough up this format.

Additionally, I am trying for a solution that does not require the
private key to be available, which means that in any case I will still
have to parse the OpenSSH public key format.

