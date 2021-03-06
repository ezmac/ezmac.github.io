---
layout: post
title:  "Benchmarking apache vs nginx sendfile"
date:   2015-12-19 11:38:37
categories: http benchmarking nginx
---

A while back I read something that said apache and nginx could hang equally as well in sendfile performance as well as reverse proxying.  I prefer nginx configurations, but I wanted to give apache a fair chance since it does serve most of the http traffic.

## Setup

### mark1
This isn't going to be super scientific, it'll be quick and dirty.  One vhost, one directory, static files only.  Files will be 4k, 1M, 10M, 100M, 1G.  The test machine is a mid 2014 macbook pro, 2.8 GHz Intel Core i7, with 16gb of ram. T o be extra unscientific, tests will be run with both programs in a docker container.  Loads will be generated by Gatling running on the host system.  This means we'll have the overhead of Virtualbox and the docker engine itself.  That said, only one container will be running at a time and both will have the same overhead.  Loads will be 100, 1000, 10000, 100000.  If 100,000 passes without issue, I'll bump it up until failure.

Mark1 was not well thought out.  After testing a few thousand clients, it became apparent that something was not capable of handling that many connections.  Thanks Apple.  Oh and linux.  I realized that there's a maximum of 65536 socket connections allowed.  Then I remembered ulimits.  And max open files was like 256.  I bumped up the max open files to 10,000 with `ulimit -n 10000` and started tweaking test parameters.  I'm not sure if it's an OSX thing, but even as root, I couldn't set max open files past 12,000.  So, I started testing 4k files at 100 users over a 2 minute run since that's how Gatling likes to run.  The test completed with a max response time of over 8s and 0 dropped requests.  Bumping up to 4k at 500 users, the container crashed.  If I recall correctly, I had done this with a standard linux/docker system, but osx/docker has to run virtualbox.  My guess is that's where the disconnect is.

This ends the trial for now.  I'll resume when I get a real os put on a Mac.

