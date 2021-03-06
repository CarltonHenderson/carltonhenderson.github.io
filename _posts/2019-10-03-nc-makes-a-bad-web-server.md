---
classes: wide
tags:
  - networking
  - http
  - netcat
  - tcp
  - docker
title: "Netcat makes a bad web server."
toc: #true
---

## Context

In order to test some firewalls I was using netcat as a really stupid webserver:

```sh
$ sudo docker run --publish 0.0.0.0:8888:8888 alpine \
nc -lk -p 8888 -v -e echo 'HTTP/1.1 200 OK
Content-Length: 1
Content-Type: text/plain

a
'
listening on [::]:8888 ...
connect to [::ffff:172.17.0.2]:8888 from [::ffff:192.168.1.5]:49832 ([::ffff:192.168.1.5]:49832)
connect to [::ffff:172.17.0.2]:8888 from [::ffff:192.168.1.5]:49867 ([::ffff:192.168.1.5]:49867)
```

## Problem

To my surprise chrome would give me this error about 1 out of 5 times:

> This site can’t be reached
>
> The connection was reset.
>
> Try:
> * Checking the connection
> * Checking the proxy and the firewall
> * Running Windows Network Diagnostics
>
> ERR_CONNECTION_RESET

My guess was eventually that that chrome is firing off requests too fast for netcat to get its
listener back up and running. It shoots a request to `GET /` and it also fires one for `GET
/favicon.ico`. Whenever I get the error above I also get a success behind the scenes on
favicon.ico. So, that means I usually get at least one request that succeeds of the two, and
sometimes I get two that succeed, but I never get just two failed requests. So netcat always works a
little but, but not completely. Smells like a race condition to me. It seems that sometimes netcat
just doesn't serve every response that comes in.

The other weird thing is that if you leave chrome sitting there long enough with the error on the
screen then eventually it will change from an error back to `Z` - the expected contents of the page.

In fact, sometimes you see a list of requests like this in the network debugger in chrome. Note
192.168.1.7 was the server under test.

| Name                 | Status   | Type     | Initiator                               | Size           | Time  |
|----------------------|----------|----------|-----------------------------------------|----------------|-------|
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:5117      | (memory cache) | 0 ms  |
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:5117      | (memory cache) | 0 ms  |
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:-Infinity | (memory cache) | 0 ms  |
| data:image/svg+xml;… | 200      | svg+xml  | chrome-error://chromewebdata/:-Infinity | (memory cache) | 0 ms  |
| 192.168.1.7          | (failed) | document | Other                                   | 0 B            | 29 ms |

Chrome will keep requesting `GET /` from 192.168.1.7, getting more failures, looking something like this:

| Name                 | Status   | Type     | Initiator                               | Size           | Time  |
|----------------------|----------|----------|-----------------------------------------|----------------|-------|
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:5117      | (memory cache) | 0 ms  |
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:5117      | (memory cache) | 0 ms  |
| data:image/png;base… | 200      | png      | chrome-error://chromewebdata/:-Infinity | (memory cache) | 0 ms  |
| data:image/svg+xml;… | 200      | svg+xml  | chrome-error://chromewebdata/:-Infinity | (memory cache) | 0 ms  |
| 192.168.1.7          | (failed) | document | Other                                   | 0 B            | 29 ms |
| 192.168.1.7          | (failed) | document | Other                                   | 0 B            | 33 ms |
| 192.168.1.7          | (failed) | document | Other                                   | 0 B            | 25 ms |
| 192.168.1.7          | (failed) | document | Other                                   | 0 B            | 24 ms |

When it does finally succeed the network list is cleared! We're left with a list looking like this:

| Name                 | Status   | Type     | Initiator                               | Size           | Time  |
|----------------------|----------|----------|-----------------------------------------|----------------|-------|
| 192.168.1.7          | 200      | document  |  Other                                 | 61 B           | 37 ms |
| favicon.ico          | 200      | text/plain | Other                                 | 61 B           | 23 ms |

## Moral

Moral of the story? Netcat isn't foolproof and Chrome isn't as simple as you might think. Apparently it will
retry connection which have been reset.

## Solution

If you want to serve a misc HTTP page and already have docker installed then just use a real web
server like this:

`docker run --publish 0.0.0.0:8888:80 nginx`
