---
title: Long polling with Javascript and Python
date: 2013-11-23
layout: post
tags:
  - python
  - javascript
  - gevent
  - bottle
---

In this post I'm going to step through an example web chat system
implemented in Python (with Bottle and gevent) that uses long polling
to implement a simple publish/subscribe mechanism for efficiently
updating connected clients in real time.

My [pubsub_example][] repository on [GitHub][] has a complete
project that implements the ideas discussed in this article.  This
project can be deployed directly on [OpenShift][] if you want to try
things out on your own.  You can also try it out online at
<http://pubsub.example.oddbit.com/>.

[openshift]: http://openshift.com/
[github]: http://github.com/
[pubsub_example]: http://github.com/larsks/pusub_example/

## What is long polling?

Long polling is a technique used in web applications to enable a
low-latency response to server messages without the CPU or traffic
overhead of high frequency polling.

A client makes a request to the web server, but rather than responding
immediately the server holds the connection (for a potentially lengthy
time), and only response when data is available.  The client will
react to this data and then restart the poll and wait for more data.

## Long polling with Jquery

I've opted to use [jquery][] as part of my client-side implementation,
because I've used it a little in the past and it simplifies things a
great deal.

There are a number of articles out there that describe the client-side
implementation of long polling using jquery.  [This][techoctave]
article from Seventh Octave gives what is a typical example:

    (function poll(){
      $.ajax({
        url: "/poll",
        success: function (data) {
          do_something_with(data);
        },
        dataType: "json",
        complete: poll,
        timeout: 30000
      });
    })();

This defines a function `poll()` that gets called automatically.
Because I'm not a JavaScript person it took me a moment to figure out
exactly how that worked.  So that you are less mysified than I: The
basic structure of this function is:

    (function poll() {...})();

Since the `function` keyword returns a value (a reference to the
defined function), we can call that reference immediately.  This
construct is entirely equivalent to:

    function poll() {...}
    poll();

When called, it fires off an asynchronous AJAX request to `/poll` on
our server.  If the server sends a response, the browser will call the
function in the `success` attribute, which will presumably do
something useful with the response data.

It's important not to gloss over the fact that the [ajax][] method is
called asynchronously.  The `poll()` function returns immediately when
it is called, allowing your client to continue processing your
script(s).

[ajax]: http://api.jquery.com/jQuery.ajax/

The callable in the `complete` attribute will be called at the end of
the AJAX request, regardless of whether or not the request was
successful. The use of the `timeout` attribute ensures that this code
will not poll more frequently than once every 30 seconds (this helps
prevent the code from monopolizing the CPU by polling too frequently
if an error is causing the AJAX request to return immediately).

[techoctave]: http://techoctave.com/c7/posts/60-simple-long-polling-example-with-javascript-and-jquery

## A simple web page

Our chat application is going be built around a very simple web page
with two fields (one for a "nickname", and one for entering messages
to send) and a container for printing messages received from the
server.  Stripped of headers and extraneous content, it looks like
this:

    <form id="chatform">
      <table>
        <tr>
          <td>
            <label for="nick">Nickname:</label>
          </td>
          <td>
            <input id="nick" size="10" />
          </td>
        </tr>
        <tr>
          <td>
            <label for="message">message:</label>
          </td>
          <td>
            <input id="message" size="40" />
            <button id="send">Send</button>
          </td>
        </tr>
      </table>
    </form>

    <hr />

    <div id="conversation"></div>

It doesn't get much simpler than that.


We'll use [jquery][] to attach JavaScript functions to various actions
using constructs such as the following, which attaches the
`send_message` to the `click` event on an element with id `send`:

    $("#send").click(send_message);

Attaching functions this way, rather than using `onclick=` attributes
in our HTML, helps keep the HTML simple.

## Making things go

For this simple application, our client is going to need to implement
two basic operations:

- Sending messages from the user to the server, and
- Receiving messages from the server and displaying them to the user.

Polling for new messages is implemented using a function that looks
very much like the sample shown above.  The final code looks like
this:

    function poll() {
            var poll_interval=0;

            $.ajax({
                    url: poll_url,
                    type: 'GET',
                    dataType: 'json',
                    success: function(data) {
                            display_message(data);
                            poll_interval=0;
                    },
                    error: function () {
                            poll_interval=1000;
                    },
                    complete: function () {
                            setTimeout(poll, poll_interval);
                    },
            });
    }

Upon successfully receiving a message from the server this starts
`poll()` again immediately, but in the event of an error this code
waits one second (1000 ms) before initiating a new poll.

The `display_message` function simply updates a `<div>` element using
the data supplied by the server:

    function display_message(data) {
            $("#conversation").append("<p><span class='nick'>"
                                     + (data['nick'] ? data['nick'] : "&lt;unknown&gt;")
                                     + "</span>: " + data['message'] + "</p>");
                                     $("#conversation").each(function () {
                                             this.scrollTop = this.scrollHeight;
                                     });
    }

Sending messages is even simpler; we just make a one-off AJAX request
to send the message to the server:

    function send_message() {
            $.ajax({
                    url: '/pub',
                    type: 'POST',
                    dataType: 'json',
                    data: {
                            nick: $("#nick").val(),
                            message: $("#message").val(),
                    },
                    complete: function () {
                            $("#message").val("");
                    },
            });
    }

This reads the value of the `nick` and `message` fields in our HTML
document and then posts the message to the server.

The complete JavaScript code can be found online as [pubsub.js][].

[pubsub.js]: https://github.com/larsks/pubsub_example/blob/master/static/pubsub.js

## Making Bottle asynchronous

I generally lean on [Bottle][] when writing simple Python web
applications.  It's simple to work with, but Bottle's native server is
single-threaded, which makes it ill suited to a long-poll scenario:
when one request is active, all other connections will block until the
first request has been serviced.  You can test this out yourself with
the following [simple webapp][]:

    import time
    import bottle

    @bottle.route('/')
    def default():
        data = [ 'one', 'two', 'three', 'four' ]
        for d in data:
            yield d
            time.sleep(5)

    def main():
        bottle.run(port=9090)

    if __name__ == '__main__':
        main()

[simple webapp]: https://github.com/larsks/pubsub_example/blob/master/example_blocking.py

Open two simultaneous connections to this application.  In two
dfferent windows, run the following command at approximately the same
time:

    curl --trace-ascii /dev/stderr http://localhost:9090

You'll see that one will block with no activity until the first
completes.

Fortunately, there are a number of solutions to this issue.  I opted
to use Bottle's support for [gevent][], an asynchronous networking
library that includes a WSGI server.  Using Bottle's `gevent` support
is easy; the above code, using the `gevent` server, would look 
[like this][]:

    import os
    import sys
    import argparse
    import time

    from gevent import monkey; monkey.patch_all()
    import bottle

    @bottle.route('/')
    def default():
        data = [ 'one', 'two', 'three', 'four' ]
        for d in data:
            yield d
            time.sleep(5)

    def main():
        bottle.run(port=9090, server="gevent")

    if __name__ == '__main__':
        main()

[like this]: https://github.com/larsks/pubsub_example/blob/master/example_nonblocking.py

The [monkey.patch_all][] routine in the above code is necessary to
patch a number of core Python libraries to work correctly with gevent.

If you re-run the `curl --trace-ascii ...` test from earlier, you'll
see that this webapp will now service multiple requests
simultaneously.

## Writing the server: receiving messages

The server needs to perform two basic operations:

- Receive a message from one client, and
- Broadcast that message to all connected clients.

Receiving a message is easy (we're just grabbing some data from a
`POST` request), but how do we handle the broadcast aspect of things?
In this application, I opted to use [0MQ][], a communication library
that has been described as "[sockets on steroids][]".  In this case,
two features of 0MQ are particularly attractive:

- support for publish/subscribe communication patterns, and
- an easy to use [in-process message transport][inproc]

[0mq]: http://zeromq.org/
[sockets on steroids]: https://speakerdeck.com/methodmissing/zeromq-sockets-on-steroids
[inproc]: http://api.zeromq.org/2-1:zmq-inproc

We'll start by creating a global 0MQ `PUB` socket (called `pubsuck`)
that will be used as one end of our in-process message bus:

    ctx = zmq.Context()
    pubsock = ctx.socket(zmq.PUB)
    pubsock.bind('inproc://pub')

With this in place, the code for receiving messages is trivial:

    @app.route('/pub', method='POST')
    def pub():
        global pubsock

        pubsock.send_json({
            'message': bottle.request.params.get('message'),
            'nick': bottle.request.params.get('nick'),
            })
        return {'status': 'sent'}

We grab the `message` and `nick` parameters from a `POST` request and
publish a JSON message onto the message bus.

## Writing the server: sending messages

Having received a message from a client, our task is to send that out
to all connected clients.  

    @app.route('/sub')
    def sub():
        bottle.response.content_type = 'application/json'
        rfile = bottle.request.environ['wsgi.input'].rfile
        return worker(rfile)

    def worker(rfile):
        subsock = ctx.socket(zmq.SUB)
        subsock.setsockopt(zmq.SUBSCRIBE, '')
        subsock.connect('inproc://pub')

        poll = zmq.Poller()
        poll.register(subsock, zmq.POLLIN)
        poll.register(rfile, zmq.POLLIN)

        while True:
            events = dict(poll.poll())

            if rfile.fileno() in events:
                break

            if subsock in events:
                msg = subsock.recv_json()
                yield(json.dumps(msg))
                break

        subsock.close()

[monkey.patch_all]: http://www.gevent.org/gevent.monkey.html
[bottle]: http://bottlepy.org/docs/
[async]: http://bottlepy.org/docs/dev/async.html
[gevent]: http://www.gevent.org/
[jquery]: http://jquery.com/
