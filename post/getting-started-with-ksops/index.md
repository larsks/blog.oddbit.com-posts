---
categories:
- tech
date: '2021-03-09'
draft: false
tags:
- openshift
- kubernetes
- ksops
- gpg
title: Getting started with KSOPS

---

## Set up GPG

### Install GPG

GPG will be pre-installed on most Linux distributions. You can check
if it's installed by running e.g. `gpg --version`. If it's not
installed, you will need to figure out how to install it for your
operating system.

### Create a key

Run the following command to create a new GPG keypair:

```
gpg --full-generate-key
```

This will step you through a series of prompts. First, select a key
type. You can just press `<RETURN>` for the default:

```
gpg (GnuPG) 2.2.25; Copyright (C) 2020 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection?
```

Next, select a key size. The default is fine:

```
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072)
Requested keysize is 3072 bits
```

You will next need to select an expiration date for your key.  The
default is "key does not expire", which is a fine choice for our
purposes. If you're interested in understanding this value in more
detail, the following articles are worth reading:

- [Does OpenPGP key expiration add to security?][expire-1]
- [How to change the expiration date of a GPG key][expire-2]

[expire-1]: https://security.stackexchange.com/questions/14718/does-openpgp-key-expiration-add-to-security/79386#79386
[expire-2]: https://www.g-loaded.eu/2010/11/01/change-expiration-date-gpg-key/

Setting an expiration date will require that you periodically update
the expiration date (or generate a new key).

```
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y
```

Now you will need to enter your identity, which consists of your name,
your email address, and a comment (which is generally left blank).
Note that you'll need to enter `o` for `okay` to continue from this
prompt.

```
GnuPG needs to construct a user ID to identify your key.

Real name: Your Name
Email address: you@example.com
Comment:
You selected this USER-ID:
    "Your Name <you@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
```

Lastly, you need to enter a password. In most environments, GPG will
open a new window asking you for a passphrase. After you've entered and
confirmed the passphrase, you should see your key information on the
console:

```
gpg: key 02E34E3304C8ADEB marked as ultimately trusted
gpg: revocation certificate stored as '/home/lars/tmp/gpgtmp/openpgp-revocs.d/9A4EB5B1F34B3041572937C002E34E3304C8ADEB.rev'
public and secret key created and signed.

pub   rsa3072 2021-03-11 [SC]
      9A4EB5B1F34B3041572937C002E34E3304C8ADEB
uid                      Your Name <you@example.com>
sub   rsa3072 2021-03-11 [E]
```

### Publish your key

You need to publish your GPG key so that others can find it. You'll
need your key id, which you can get by running `gpg -k --fingerprint`
like this (using your email address rather than mine):

```
$ gpg -k --fingerprint lars@redhat.com
```

The output will look like the following:

```
pub   rsa4096/0x7BF065CD97112F06 2013-09-12 [SC] [expires: 2022-10-04]
      Key fingerprint = 5B2E 9490 ED4F 7BB3 79EC  93C1 7BF0 65CD 9711 2F06
uid                   [ultimate] Lars Kellogg-Stedman <lars@redhat.com>
uid                   [ultimate] Lars Kellogg-Stedman <lkellogg@redhat.com>
uid                   [ultimate] Lars Kellogg-Stedman <larsks@redhat.com>
sub   rsa4096/0x2CDCB13F055186A4 2013-09-12 [E]
sub   rsa4096/0xE4D8B2B13F5F9E29 2013-09-12 [S] [expires: 2022-10-04]
sub   rsa4096/0xB4FDC7573F885287 2013-09-12 [A] [expires: 2022-10-04]
```

Look for the `Key fingerprint` line, you want the value after the `=`.
Use this to publish your key to `keys.openpgp.org`:


```
gpg --keyserver keys.opengpg.org \
  --send-keys '5B2E 9490 ED4F 7BB3 79EC 93C1 7BF0 65CD 9711 2F06'
```

You will shortly receive an email to the address in your key asking
you to approve it. Once you have approved the key, it will be
published on <https://keys.openpgp.org> and people will be able to look
it up by address or key id.  For example, you can find my public key
at <https://keys.openpgp.org/vks/v1/by-fingerprint/5B2E9490ED4F7BB379EC93C17BF065CD97112F06>.

## The operate-first toolbox

The [operate-first][] project provides an [operate-first toolbox
container][opf-toolbox] with all the necessary tools pre-installed.
This requires that you have a recent version of the [toolbox][]
command installed; if you're on Fedora 33 (or later) you should be all
set.

[operate-first]: https://www.operate-first.cloud/
[opf-toolbox]: https://quay.io/repository/operate-first/opf-toolbox?tag=latest&tab=tags
[toolbox]: https://github.com/containers/toolbox

If you're in an environment in which the `toolbox` command isn't
available, or you just like to understand all the steps yourself,
skip to the next section.

To run the operate-first toolbox, first create a toolbox container:

```
$ toolbox create --image quay.io/operate-first/opf-toolbox:v0.3.2
Created container: opf-toolbox
Enter with: toolbox enter --container opf-toolbox-v0.3.2
```

And then enter the container:

```
$ toolbox enter --container opf-toolbox-v0.3.2
```

Inside the container, you will have access to your local files and
directories as well as the `kustomize` command, already configured to
make use of the KSOPS plugin.

## Installing Kustomize and KSOPS

In this section, we'll get all the necessary tools installed on your
system in order to interact with a repository using Kustomize and
KSOPS.

### Install Kustomize

Pre-compiled binaries of Kustomize are published [on
GitHub][gh-kustomize]. To install the command, navigate to the current
release ([v4.0.5][] as of this writing)  and download the appropriate
tarball for your system. E.g, for an x86-64 Linux environment, you
would grab [kustomize_v4.0.5_linux_amd64.tar.gz][].

[gh-kustomize]: https://github.com/kubernetes-sigs/kustomize/releases
[v4.0.5]: https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv4.0.5
[kustomize_v4.0.5_linux_amd64.tar.gz]: https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.0.5/kustomize_v4.0.5_linux_amd64.tar.gz

The tarball contains a single file. You need to extract this file and
place it somwhere in your `$PATH`.  For example, if you use your
`$HOME/bin` directory, you could run:

```
tar -C ~/bin -xf kustomize_v4.0.5_linux_amd64.tar.gz
```

Or to install into `/usr/local/bin`:

```
sudo tar -C /usr/local/bin -xf kustomize_v4.0.5_linux_amd64.tar.gz
```

Run `kustomize` with no arguments to verify the command has been
installed correctly.

### Install sops

The KSOPS plugin relies on the [sops][] command, so we need to install
that first. Binary releases are published on GitHub, and the current
release is [v3.6.1][].

Instead of a tarball, the project publishes the raw binary as well as
packages for a couple of different Linux distributions. For
consistency with the rest of this post we're going to grab the [raw
binary][sops-v3.6.1.linux]. We can install that into `$HOME/bin` like this:

```
curl -o ~/bin/sops https://github.com/mozilla/sops/releases/download/v3.6.1/sops-v3.6.1.linux
chmod 755 ~/bin/sops
```

[sops]: https://github.com/mozilla/sops
[v3.6.1]: https://github.com/mozilla/sops/releases/tag/v3.6.1
[sops-v3.6.1.linux]: https://github.com/mozilla/sops/releases/download/v3.6.1/sops-v3.6.1.linux

### Install KSOPS

KSOPS is a Kustomize plugin. The `kustomize` command looks for plugins
in subdirectories of `$HOME/.config/kustomize/plugin`. Directories are
named after an API and plugin name. In the case of KSOPS, `kustomize`
will be looking for a plugin named `ksops` in the
`$HOME/.config/kustomize/plugin/viaduct.ai/v1/ksops/` directory.

The current release of KSOPS is [v2.4.0][], which is published as a
tarball. We'll start by downloading
[ksops_2.4.0_Linux_x86_64.tar.gz][], which contains the following
files:

```
LICENSE
README.md
ksops
```

To extract the `ksops` command to `$HOME/bin`, you can run:

```
tar -C ~/.config/kustomize/plugin/viaduct.ai/v1/ksops -xf ksops_2.4.0_Linux_x86_64.tar.gz ksops
```

[v2.4.0]: https://github.com/viaduct-ai/kustomize-sops/releases/tag/v2.4.0
[ksops_2.4.0_Linux_x86_64.tar.gz]: https://github.com/viaduct-ai/kustomize-sops/releases/download/v2.4.0/ksops_2.4.0_Linux_x86_64.tar.gz


## Test it out

Let's create a simple Kustomize project to make sure everything is
installed and functioning.

Start by creating a new directory and changing into it:

```
mkdir kustomize-test
cd kustomize-test
```

Create a `kustomization.yaml` file that looks like this:

```
generators:
  - secret-generator.yaml
```

Put the following content in `secret-generator.yaml`:

```
---
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
files:
  - example-secret.enc.yaml
```

This instructs Kustomize to use the KSOPS plugin to generate content
from the file `example-secret.enc.yaml`.

Configure `sops` to use your GPG key by default by creating a
`.sops.yaml` (note the leading dot) similar to the following (you'll
need to put your GPG key fingerprint in the right place):

```
creation_rules:
  - encrypted_regex: "^(users|data|stringData)$"
    pgp: <YOUR KEY FINGERPRINT HERE>
```

The `encrypted_regex` line tells `sops` which attributes in your YAML
files should be encrypted. The `pgp` line is a (comma delimited) list
of keys to which data will be encrypted.

Now, edit the file `example-secret.enc.yaml` using the `sops` command.
Run:

```
sops example-secret.enc.yaml
```

This will open up an editor with some default content. Replace the
content with the following:


```
apiVersion: v1
kind: Secret
metadata:
    name: example-secret
type: Opaque
stringData:
    message: this is a test
```

Save the file and exit your editor. Now examine the file; you will see
that it contains a mix of encrypted and unencrypted content. When
encrypted with my private key, it looks like this:

```
$ cat example-secret.enc.yaml
apiVersion: v1
kind: Secret
metadata:
    name: example-secret
type: Opaque
stringData:
    message: ENC[AES256_GCM,data:yOb7WAVQMU/m2d5ON2M=,iv:g4R9QPPjV59s7UTCc0tuwMVDYZmxcxXkFM4T/7E3sIY=,tag:vmmJrRF5G3cNatofb+MdqA==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    lastmodified: '2021-03-11T17:15:02Z'
    mac: ENC[AES256_GCM,data:g9eFG6A1qSpjYUxt3w98jPn4eEPmDN1AdX+5OYik8Al6O0WlO1vRF/2BhQN+t26VBccaYOyxDCay+Qrm1Im0DxjdIFqg6thYyl+hvdSsNZXpHWwK7mcnhfAqFa9rVeQBS0DBJvGtxn0R9QBydxlb3bdnfT6KvpUgo2799cJkujI=,iv:BIrocTilbkyEdcneaX9cBuu+MPSJkkF0ZOcsS3D54MM=,tag:RT5Wv6BfpoCR5mUEJ/lfiQ==,type:str]
    pgp:
    -   created_at: '2021-03-11T17:13:40Z'
        enc: |-
            -----BEGIN PGP MESSAGE-----

            wcFMAyzcsT8FUYakARAAkkjoaHZ5HR1CYduoHyF5YlJOTKlNQWnZ5JiDDHWKM3Ef
            cD++XC8P4tpNzBSqwOswjdp0ENYEcLIU5tT2TO1FJf+NSHSGVYx3BM39eTbFRaKI
            s/vmHM9Q7iE1ms7S7x+3cZkckHcxrsF0YOBEYaIeAxXNx95B1Q2PPYjHgnVrOs/f
            MCxWZF1G8wUcFGB1Sz3rK7mP1B5Bl6S6ZY7DrppVrN9xfSYpk+qHp4O8nO3OB8AG
            saGlKuEY44JofoCyHTWDSFyrLcc3njLZw72vXDUnXVYPGCOZsUTGhmIxKrcgbLky
            xXSHromwNjVbbfXDYaIGw2ZSaKw5fXDycuTuH/EyDNEtO17/X69aILcqSZaw/Hdj
            PGnMLhxhzc7ZH7CW2aNcpb+NfpNOP2q9WXI3zeqbSuGiLQMMCKFH6EIIXR6boRdU
            xDHczYIwDtiTmhidCqYsiRj0R5tlAW6VEvX2ecLGJ3Bh1rdFjRMJQ7/dXY/2y+3B
            NitNU9hmHjZuduNnUu9cDPJNV/j1YQk936pmLqlSettWuQvk8LRQyz1Dq+Xh2liS
            qVbxFYnmciKThWdEoVo0q8bwo6lchINvA8THqFHSHk95apFAhxOiv+yxalDG+a+Y
            dSK6xY6T3rcGzyhNZxw2tuvD83LPwralo/eIj5f7o57h+3YIQ8+ORAL2+X6qGFnS
            4AHkfiCrp5zM0Wuwe0shHcef0uH7+eDs4NbhshDgB+KtHzoF4JPlDujhm9j3gqi9
            dCN4cRPG010hzKQ84ZKeMvYYShz+np7gPeTsTlLP4hKgjUd3nQaCejJ64vGG5k3h
            T5YA
            =b7Aw
            -----END PGP MESSAGE-----
        fp: 5B2E9490ED4F7BB379EC93C17BF065CD97112F06
    encrypted_regex: ^(users|data|stringData)$
    version: 3.6.1
```

Finally, attempt to render the project with Kustomize by running:

```
kustomize build --enable-alpha-plugins
```

This should produce on stdout the unencrypted content of your secret:

```
apiVersion: v1
kind: Secret
metadata:
    name: example-secret
type: Opaque
stringData:
    message: this is a test
```
