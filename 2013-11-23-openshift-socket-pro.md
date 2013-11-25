---
title: OpenShift socket problems: it's not always about websocket
date: 2013-11-23
layout: post
tags:

---

This is a followup to my [previous post][] regarding long-poll
servers and Python.  Once I had everything working on my laptop I
uploaded it to [OpenShift][]...only to find that while it still worked
correctly, the number of open connections was climbing actively.  What
was going on?

I finally found [this post][pass-websockets] by Marak Jelen discussing
issues with [websockets][] in OpenShift, which says, among other
things:

> For OpenShift as a PaaS provider, WebSockets were a big challenge.
> The routing layer that sits between the user's browser and your
> application must be able to route and handle WebSockets. OpenShift
> uses Apache as a reverse proxy server and a main component to route
> requests throughout the platform. However, Apache's mod_proxy has
> been problematic with WebSockets, so OpenShift implemented a new
> Node.js based routing layer that provides scalability and the
> possibility to expand features provided to our users.

In order to work around these problems, an alternate [Node.js][] based
front-end has been deployed on port 8000.  So if your application is
normally available at `http://myapplication-myname.rhcloud.com`, you
can also access it at `http://myapplication-myname.rhcloud.com:8000`.

Not unexpectedly, it seems that the same things that can cause
difficulties with WebSockets connections can also interfere with the
operation of a long-poll server.  The root of the problem is that your
service running on OpenShift never receives notifications of client
disconnects...you can see this by opening up a connection to your
service:

  $ socat - tcp:myapplication-myname.rhcloud.com:80
  GET / HTTP/1.1
  Host: myapplication-myname.rhcloud.com

Leave the connection open and [log in to your OpenShift instance][login].  Run `netstat` to see the existing connection:

    $ netstat -tln | grep $OPENSHIFT_PYTHON_PORT | grep ESTABLISHED
    tcp        0      0 127.6.26.1:15368            127.6.26.1:8080             ESTABLISHED 
    tcp        0      0 127.6.26.1:8080             127.6.26.1:15368            ESTABLISHED 

Now close your client, and re-run the `netstat` command on your
OpenShift instances.  You will find that the client connection  from
the front-end proxies to your server is still active.  Because the
server never receives any notification that the client has closed the
connection, no amount of `select` or `poll` or anything else will
solve this problem.

Now, try the same experiment using port 8000.

After opening the poll connection:

    tcp        0      0 127.6.26.1:8080             127.6.26.1:15685            ESTABLISHED 
    tcp        0      0 127.6.26.1:15685            127.6.26.1:8080             ESTABLISHED 

After closing the client, this connection is no longer evident.  This
means we can handle client disconnections properly with something a
simple `select` or `poll` operation, like this:

def worker(q, rfile):

    s = ctx.socket(zmq.SUB)
    s.setsockopt(zmq.SUBSCRIBE, '')
    s.connect(pub_socket_uri)

    poll = zmq.Poller()
    poll.register(s, zmq.POLLIN)
    poll.register(rfile, zmq.POLLERR|zmq.POLLIN)

    while True:
        events = dict(poll.poll())

        # If the client disconnects there will be a read event on the
        # socket.
        if rfile.fileno() in events:
            break

        # If there was a message on the message bus, process it.
        If s in events:
            msg = s.recv_json()
            handle_msg(msg)
            break

    s.close()
    q.put(StopIteration)

BUT OMG IT STILL DOESN'T WORK WHY?

Because javascript cross-domain security policies.

[previous post]: {% post_url 2013-11-23-long-polling-with-ja %}
[openshift]: http://www.openshift.com/
[paas-websockets]: https://www.openshift.com/blogs/paas-websockets
[websockets]: http://en.wikipedia.org/wiki/WebSocket
[login]: https://www.openshift.com/developers/remote-access

