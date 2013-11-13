---
layout: post
title: Rate limiting made simple
date: 2011-12-26
tags: networking,linux
---

I use [CrashPlan][1] as a backup service. It works and is very simple to set
up, but has limited options for controlling bandwidth. In fact, if you're
running it on a headless system (e.g., a fileserver of some sort), your options
are effectively "too slow" and "CONSUME EVERYTHING". There is an [open
request][2] to add time-based limitations to the application itself, but for
now I've solved this using a very simple traffic shaping configuration. 
Because the learning curve for "tc" and friends is surprisingly high, I'm
putting my script here in the hopes that other people might find it useful, and
so that I can find it when I need to do this again someday. 

<script src="https://gist.github.com/4014881.js"></script>

 [1]: http://www.crashplan.com/
 [2]: https://crashplan.zendesk.com/entries/446273-throttle-bandwidth-by-hours?page=1#post_20799486

