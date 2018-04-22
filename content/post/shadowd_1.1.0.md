+++
title = "Release of shadowd 1.1.0"
description = ""
tags = [
    "release",
    "shadowd",
    "shadowd_ui",
    "performance",
]
date = "2015-03-28"
categories = [
    "Development",
    "Shadow Daemon",
]
menu = "main"
url = "/release-shadowd_1.1.0/"
aliases = [
    "/post/shadowd_1.1.0/",
]
author = "Hendrik Buchwald"
+++

It is my pleasure to announce the release of *shadowd 1.1.0* as well as *shadowd_ui 1.1.0* of the [Shadow Daemon](https://shadowd.zecure.org/) web application firewall. This update improves the performance, attack detection and ease of use. There are five major changes:

 * A native flood protection. It is no longer necessary to use *fail2ban* to prevent flooding of the logs, it happens automatically now.
 * A storage queue. This removes a huge bottleneck from Shadow Daemon, the permanent storage of requests.
 * Optimizations of the database layout to improve the performance.
 * New blacklist filters/signatures to detect more attacks, e.g. *shellshock*, *cross-site scripting*, *server-site includes* and *code evaluation*.
 * An option for the whitelist rules generator to automatically unify arrays. This makes it much easier to generate rules for big web applications.

There are no new major additions, but this update does improve the overall experience a lot, so I highly recommend to apply it. Most changes are based on feedback, so keep it coming :)

<!--more-->

## Performance

Talk is cheap, here are some actual benchmarks. The target is the front page of a Wordpress 4.0 installation with four posts. Note the *requests per second*.

### Unprotected

```
dev:~$ ab -n 5000 -c 50 http://phpconnector/wordpress/

Server Software:        lighttpd/1.4.33
Server Hostname:        phpconnector
Server Port:            80

Document Path:          /wordpress/
Document Length:        41048 bytes

Concurrency Level:      50
Time taken for tests:   59.356 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      206390000 bytes
HTML transferred:       205240000 bytes
Requests per second:    84.24 [#/sec] (mean)
Time per request:       593.563 [ms] (mean)
Time per request:       11.871 [ms] (mean, across all concurrent requests)
Transfer rate:          3395.64 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       1
Processing:   165  591  34.3    586     884
Waiting:      145  573  33.9    568     852
Total:        167  591  34.3    586     885

Percentage of the requests served within a certain time (ms)
  50%    586
  66%    592
  75%    597
  80%    600
  90%    615
  95%    639
  98%    700
  99%    725
 100%    885 (longest request)
```

### Protected

```
dev:~$ ab -n 5000 -c 50 http://phpconnector/wordpress/

Server Software:        lighttpd/1.4.33
Server Hostname:        phpconnector
Server Port:            80

Document Path:          /wordpress/
Document Length:        41048 bytes

Concurrency Level:      50
Time taken for tests:   85.559 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      206390000 bytes
HTML transferred:       205240000 bytes
Requests per second:    58.44 [#/sec] (mean)
Time per request:       855.590 [ms] (mean)
Time per request:       17.112 [ms] (mean, across all concurrent requests)
Transfer rate:          2355.72 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       1
Processing:   100  851  96.1    819    1246
Waiting:       70  832  94.6    800    1216
Total:        101  851  96.1    819    1247

Percentage of the requests served within a certain time (ms)
  50%    819
  66%    840
  75%    868
  80%    921
  90%   1000
  95%   1036
  98%   1087
  99%   1128
 100%   1247 (longest request)
```

Without protection the server processes 84.24 requests per second, with protection it processes 58.44 requests per second. This means Shadow Daemon decreases the performance in this test by about 41%. That is not bad, not bad at all, because if you do not have a website with thousands of visitors per second 41% will not be noticeable.

<img src="/img/performance_pie.svg" title="Requests per second" style="width:100%" />

If you do have a high traffic site you have to decide between: a bit less performance and high security or default performance and no security at all.

The storage queue and flooding protection that are introduced in this update make sure that the performance is not impaired by large amounts of attacks that have to be stored permanently. The storage queue simply decouples analysis of requests and storage, so the result of the analysis can be send back to the connector even if the request is not stored yet. The flood protection on the other hand prevents the obstruction of the queue by single clients.

**More information at https://shadowd.zecure.org/**
