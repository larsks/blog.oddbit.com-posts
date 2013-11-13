---
layout: post
title: A systemd unit for ucarp
date: 2013-02-21
---

In Fedora 17 there are still a number of services that either have not
been ported over to `systemd` or that do not take full advantage of
`systemd`.  I've been investigating some IP failover solutions
recently, including [ucarp][], which includes only a System-V style
init script.

I've created a [template service][template] for ucarp that will let
you start a specific virtual ip like this:

    systemctl start ucarp@001

This will start ucarp using settings from `/etc/ucarp/vip-001.conf`.
The unit file is [on github][github] and embedded here for your
reading pleasure:

<script src="https://gist.github.com/larsks/5009872.js"></script>

[ucarp]: http://www.pureftpd.org/project/ucarp
[template]: http://0pointer.de/blog/projects/instances.html
[github]: https://gist.github.com/larsks/5009872

