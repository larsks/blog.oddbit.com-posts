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

## What is long polling?

Long polling is a technique used in web applications to enable a
low-latency response to server messages without the CPU or traffic
overhead of high frequency polling.

A client makes a request to the web server, but rather than responding
immediately the server holds the connection (for a potentially lengthy
time), and only response when data is available.  The client will
react to this data and then restart the poll and wait for more data.

## Long polling in the client

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

## Making Bottle asynchronous

[example]: http://github.com/larsks/pusub_example/
[example-online]: http://pubsub-oddbit.rhcloud.com/
[bottle]: http://bottlepy.org/docs/
[async]: http://bottlepy.org/docs/dev/async.html
[gevent]: http://www.gevent.org/
[jquery]: http://jquery.com/

