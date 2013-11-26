---
title: Debugging Neutron
date: 2013-11-25
layout: post
tags:
  - openstack
  - neutron
  - sdn
---

## My instance isn't getting an ip address

### Boot a Cirros instance

Boot an instance of the [Cirros][] image on your compute host.  The
Cirros is small, has all the tools we need for debugging network
problems, and will let you log in on the console.

[cirros]: https://launchpad.net/cirros/+download

### Generate some traffic

Start by generating some traffic:

    # udhcpc -f -t 1000 -i eth0

[neutron-architecture]: /assets/quantum-gre.svg

