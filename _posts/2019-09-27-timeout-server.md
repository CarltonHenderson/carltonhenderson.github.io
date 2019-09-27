# Simulating Slow HTTP Server and Client Timeouts

Here's yet anothe way to use `netcat` (a.k.a. `nc`), this time to simulate a *very* slow web server
(or any TCP server). Netcat will listen on port 8888 and any time a client connects it will respond
with a live stream of the output of `sleep 60s` - a very slow nothing.

## Server

```sh
nc -lk -p 8888 -e sleep 60s
```

This will run until interrupted with `ctrl+c`

### Uncooperative versions of Netcat

You may get the following error if your version of `nc` doesn't support `-lk` or `-e PROG`. These
options makes the nc server persistent, whereas by default it will quit after serving its first
client.

```sh
60s: forward host lookup failed: Unknown host
```

A solution, if you have docker installed, is to simply use the netcat that comes with the alpine docker
image, remembering to publish the portlier

```sh
docker run -p127.0.0.1:8888:8888 alpine nc -lk -p 8888 -e sleep 60s
```

## Example Client Behavior

### Patient Client - netcat

With the server running we can see it is working as intended by again using `nc` - as a client this time.

```sh
$ time nc 127.0.0.1 8888

real    1m0.057s
user    0m0.000s
sys     0m0.000s
```

### Impatient Client - ruby

Some clients, such as ruby in its default mode, will timeout after an amount of time they choose.

```sh
06:16 $ time ruby -e "require 'open-uri'; open('http://localhost:8888').read"
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
the word on the street that ruby has a 1 minute timeout. Perhaps it retries because they response
from `nc` was completely empty.

```sh
$ sudo docker run -p127.0.0.1:8888:8888 alpine nc -lk -p 8888 -e sleep 600s -v
listening on [::]:8888 ...
connect to [::ffff:172.17.0.2]:8888 from [::ffff:172.17.0.1]:37194 ([::ffff:172.17.0.1]:37194)
connect to [::ffff:172.17.0.2]:8888 from [::ffff:172.17.0.1]:37928 ([::ffff:172.17.0.1]:37928)
```

### Dilligent Client - wget

Like Ruby, wget will retry if it receives no data in response to a request, but it is much more
diligent. I interrupted it before it finished its default limit of 20 retries.

```sh
$ wget 127.0.0.1:8888
--2019-09-27 05:26:38--  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 05:26:45--  (try: 2)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 05:26:53--  (try: 3)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 05:27:02--  (try: 4)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 05:27:12--  (try: 5)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
Retrying.

--2019-09-27 05:27:23--  (try: 6)  http://127.0.0.1:8888/
Connecting to 127.0.0.1:8888... connected.
HTTP request sent, awaiting response... No data received.
```

