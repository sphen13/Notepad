---
layout: post
title: "Munki Proxy - Update"
description:
headline:
modified: 2017-12-30
category: software
tags: [docker, munki, nginx]
imagefeature: munki-proxy.gif
mathjax:
chart:
comments: true
featured: true
---

So after running my [munki-proxy](https://hub.docker.com/r/sphen/munki-proxy/) container for awhile and doing some more testing, I ran into some curious things.  When the cache was filling a request for the first time the download was rather slow (~3 MB/sec) even though my internet connection is capable of ~ 12 MB/sec.  While serving any cached content everything was fast and easily saturated the LAN connection.

I went back to the nginx config and made some tweaks which have improved performance.  I do wonder however if this could have a bag effect on really slow internet connections.

I have updated my [original post](/software/munki-proxy) with these changes.
{: .notice}

## Changes

So I went back and researched the nginx configuration documentation a bit.  Some of these changes here may not directly effect the performance but I decided they should have been set to begin with.

- Changed: **worker_processes** from **6 -> 1**
  - Due to the amount of clients that I expect someone using munki-proxy for, having one worker_process is more appropriate.
- Changed: **worker_connections** from **768 -> 1024**
  - Basically setting this to default
- Added: **use epoll** and **multi_accept on**
  - Documented as using I/O more efficiently
- Changed: **sendfile on** from **off -> on**
  - This had been disabled as using sendfile within docker in VirtualBox can cause issues - this is not our use-case.  This optimizes serving static files.
- Added: gzip parameters
  - **gzip_min_length 1000**
  - **gzip_types text/html application/x-javascript text/css application/javascript text/javascript text/plain text/xml**
  - This keeps gzip enabled but makes sure nginx only uses it for larger compressible files.

Now, while those are small tweaks that I thought were good to make, I did not expect them to fix my observed problem.  The real fix came down to the **slice size**.

Previously we were using a slice size of *2m* even though the docs had mentioned using *1m*.  As munki-proxy is mostly used for larger than normal web files, I had always thought that a 2 meg slice size was small - but I was going off more of the documentation that stated that you want to choose a size that can be served to the client within 1-2 seconds.  I calculated that with a slower 10 Mbit connection this would be good.  But how often do we really have connections that slow anymore?  And in the case of a OS update we are talking a very large amount of cached slices - which would be inefficient overall.  So.....  I changed it to *16m*.

- Changed: **slice** from **2m -> 16m**

I also added a lock timeout in case of a slower connection.  By default it is set to 5 seconds.  If the timeout is reached the nginx will send another request upstream which I do not want - so I just set this to 5 minutes.

- Added: **proxy_cache_lock_timeout  5m**

## Result

Now when re-creating the munki-proxy container initial cache-fill requests are as fast as my internet connection.  I am happy for now :)
