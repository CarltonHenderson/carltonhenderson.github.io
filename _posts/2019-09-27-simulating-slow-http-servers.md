---
title: "Simulating a Slow HTTP Server"
toc: true
classes: #wide
tags:
  - linux
  - networking
  - netcat
  - tcp
  - docker
---

One of the API servers at Acme Mallets is occasionally taking a very long time to respond and our
code needs to be ready for that issue and to eventually give up and fail gracefully when there's no
hope of Acme replying with the data we need. Unfortunately, we can't keep hitting the Mallet server
to test our code - they're slow because they're already under a lot of load - so how can we quickly
test our code against a slow mock server?

This calls for `netcat` (a.k.a. `nc` or the TCP/IP Swiss Army Knife), this time we'll be using it to
simulate a *very* slow web server.

## Server

Invoked with the following options, Netcat will listen on port 8888 and any time a client
connects it will respond with a live stream of the output of `sleep 60s` - a very slow nothing.

```sh
nc -lk -p 8888 -e sleep 60s
```

This will run until interrupted with `ctrl+c`

### Uncooperative versions of Netcat

We may get the following error if our version of `nc` doesn't support `-lk` or `-e PROG`. We need
these options because they make the netcat server persistent, whereas by default it will quit after
serving its first client.

```sh
60s: forward host lookup failed: Unknown host
```

A solution, since we have docker installed, is to use the version of netcat which comes with
the _alpine_ docker image, remembering to publish the port.

```sh
docker run --publish 0.0.0.0:8888:8888 alpine nc -lk -p 8888 -e sleep 60s
```

An earlier version of this blog had 127.0.0.1 in the command above. Thus docker would only list on
localhost and computers outside of yours wouldn't be able to access the netcat server.{: .notice--info}

## Example Client Behavior

### Hung/Patient Client - netcat

With the mock server running we can see an example of a "hung" client by using `nc` again - as a
client this time.

```sh
$ time nc 127.0.0.1 8888

real    1m0.057s
user    0m0.000s
sys     0m0.000s
```

### Crashed/Impatient Client - ruby

Some clients, such as ruby in its default mode, will timeout after an amount of time they choose.

```sh
$ time ruby -e "require 'open-uri'; open('http://localhost:8888').read"
/usr/lib/ruby/2.3.0/net/protocol.rb:158:in `rbuf_fill': Net::ReadTimeout (Net::ReadTimeout)
        from /usr/lib/ruby/2.3.0/net/protocol.rb:136:in `readuntil'
        from /usr/lib/ruby/2.3.0/net/protocol.rb:146:in `readline'
        from /usr/lib/ruby/2.3.0/net/http/response.rb:40:in `read_status_line'
        from /usr/lib/ruby/2.3.0/net/http/response.rb:29:in `read_new'
        from /usr/lib/ruby/2.3.0/net/http.rb:1437:in `block in transport_request'
        from /usr/lib/ruby/2.3.0/net/http.rb:1434:in `catch'
        from /usr/lib/ruby/2.3.0/net/http.rb:1434:in `transport_request'
        from /usr/lib/ruby/2.3.0/net/http.rb:1407:in `request'
        from /usr/lib/ruby/2.3.0/open-uri.rb:325:in `block in open_http'
        from /usr/lib/ruby/2.3.0/net/http.rb:853:in `start'
        from /usr/lib/ruby/2.3.0/open-uri.rb:319:in `open_http'
        from /usr/lib/ruby/2.3.0/open-uri.rb:737:in `buffer_open'
        from /usr/lib/ruby/2.3.0/open-uri.rb:212:in `block in open_loop'
        from /usr/lib/ruby/2.3.0/open-uri.rb:210:in `catch'
        from /usr/lib/ruby/2.3.0/open-uri.rb:210:in `open_loop'
        from /usr/lib/ruby/2.3.0/open-uri.rb:151:in `open_uri'
        from /usr/lib/ruby/2.3.0/open-uri.rb:717:in `open'
        from /usr/lib/ruby/2.3.0/open-uri.rb:35:in `open'
        from -e:1:in `<main>'

real    2m0.182s
user    0m0.052s
sys     0m0.012s
```

By turning on `-v`erbose logging on the server side we can see ruby tried twice - which jives with
the word on the street that ruby has a 1 minute default timeout. Perhaps it retries because they
response from `nc` was completely empty.

```sh
$ docker run -p127.0.0.1:8888:8888 alpine nc -lk -p 8888 -e sleep 600s -v
listening on [::]:8888 ...
connect to [::ffff:172.17.0.2]:8888 from [::ffff:172.17.0.1]:37194 ([::ffff:172.17.0.1]:37194)
connect to [::ffff:172.17.0.2]:8888 from [::ffff:172.17.0.1]:37928 ([::ffff:172.17.0.1]:37928)
```

### Intelligent/Dilligent Client - wget

Finally, we look at wget. Like Ruby, wget will retry if it receives no data in response to a
request, but it is much more diligent and it fails with a human-readable message instead of a
stack-trace.

```sh
✘-4 ~
06:27 $ wget 127.0.0.1:8888
--2019-09-27 06:27:22--  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:25--  (try: 2)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:29--  (try: 3)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:34--  (try: 4)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:40--  (try: 5)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:47--  (try: 6)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:27:55--  (try: 7)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:28:04--  (try: 8)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:28:14--  (try: 9)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:28:25--  (try:10)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:28:37--  (try:11)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:28:49--  (try:12)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:29:01--  (try:13)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:29:14--  (try:14)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:29:26--  (try:15)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:29:38--  (try:16)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:29:50--  (try:17)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:30:02--  (try:18)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:30:14--  (try:19)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 06:30:26--  (try:20)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Giving up.
```
