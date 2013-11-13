---
layout: post
title: Waiting for networking using PowerShell
date: 2012-11-04
tags:
  - powershell
  - networking
  - windows
---

I've recently been exploring the world of Windows scripting, and I ran
into a small problem: I was running a script at system startup, and
the script was running before the network interface (which was using
DHCP) was configured.

There are a number of common solutions proposed to this problem:

- Just wait for some period of time.

    This can work but it's ugly, and because it doesn't actually
    verify the network state it can result in things breaking if some
    problem prevents Windows from pulling a valid DHCP lease.
- Use ping to verify the availability of some remote host.

    This works reasonably well if you have a known endpoint you can
    test, but it's hard to make a generic solution using this method.

What I really wanted to do was to have my script wait until a default
gateway appeared on the system (which would indicate that Windows had
successfully acquired a DHCP lease and had configured the interface).

My first attempts involved traditional batch scripts (`.bat`) running
some variant of `route print` and searching the output.  This can
work, but it's ugly, and I was certain there must be a better way.  I
spent some time learning about accessing network configuration
information using PowerShell, and I came up with [the following
code][gist-4011808]:

<script src="https://gist.github.com/4011808.js"></script>

This uses various sort of filtering to get a list of DHCP-enabled
interfaces that have a default gateway (the `DefaultIPGateway`
attribute).  It will poll the state of things once/second up to `$tries`
times, and if nothing is available it will ultimately throw an
exception.

[gist-4011808]: https://gist.github.com/4011808

