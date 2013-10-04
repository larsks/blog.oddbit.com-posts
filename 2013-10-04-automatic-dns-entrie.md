Title: Automatic DNS entries for libvirt domains
Date: 2013-10-04
Tags: libvirt,virtualization

Have you ever wished that you could use `libvirt` domain names as
hostnames?  So that you could do something like this:

    $ virt-install -n anewhost ...
    $ ssh clouduser@anewhost

Since this is something that would certainly make my life convenient,
I put together a small script called [virt-hosts][] that makes this
possible.

Run by itself, with no options, `virt-hosts` will scan through your
running domains for interfaces on the libvirt `default` network, look
up those MAC addresses up in the corresponding `default.leases` file,
and then generate a hosts file on `stdout` like this:

    $ virt-hosts
    192.168.122.221	compute-tmp0-net0.default.virt compute-tmp0.default.virt
    192.168.122.101	centos-0-net0.default.virt centos-0.default.virt
    192.168.122.214	controller-tmp-net0.default.virt controller-tmp.default.virt

Each address will be assigned the name
`<domain_name>-<interface_name>.<network_name>.virt`.  The first
interface on the network will also be given the alias
`<domain_name>.<network_name>.virt`, so a host with multiple
interfaces on the same network would look like this:

    $ virt-hosts
    192.168.122.221	host0-net0.default.virt host0.default.virt
    192.168.122.110	host0-net1.default.virt

Of course, this is only half the solution: having generated a hosts
file we need to put it somewhere where your system can find it.

Using NetworkManager + dnsmasq
------------------------------

I am running NetworkManager with the `dnsmasq` dns plugin. I created
the file `/etc/NetworkManager/dnsmasq.d/virthosts` containing:

    addn-hosts=/var/lib/libvirt/dnsmasq/default.addnhosts

This will cause the `dnsmasq` process started by `NetworkManager` to
use that file as an additional hosts file.  I then installed the
`incron` package and dropped the following in
`/etc/incron.d/auto-virt-hosts`:

    /var/lib/libvirt/dnsmasq/default.leases IN_MODIFY /usr/local/bin/virt-hosts -u

This has `incron` listen for changes to the `default.leases` file, and
whenever it receives the `IN_MODIFY` event it runs `virt-hosts` with
the `-u` (aka `--update`) flag.  This causes `virt-hosts` to send
output to `/var/lib/libvirt/dnsmasq/default.addnhosts` instead of
`stdout`.

Using /etc/hosts
----------------

If you're not using `dnsmasq`, you could put the following into a
script called `update-virt-hosts` and run that via `incron` instead:

    #!/bin/sh

    sed -i '/^# BEGIN VIRT HOSTS/,/^# END VIRT HOSTS/ d' /etc/hosts
    cat <<EOF >>/etc/hosts
    # BEGIN VIRT HOSTS
    $(virt-hosts)
    # END VIRT HOSTS
    EOF

[virt-hosts]: https://raw.github.com/larsks/virt-utils/master/virt-hosts
[virt-utils]: https://raw.github.com/larsks/virt-utils/
[nm-using-dnsmasq]: 
