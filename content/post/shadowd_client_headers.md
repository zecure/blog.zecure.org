+++
title = "Shadow Daemon and Client Headers"
description = ""
tags = [
    "shadowd",
    "configuration",
]
date = "2020-09-02"
categories = [
    "Development",
    "Shadow Daemon",
]
menu = "main"
url = "/shadowd_client_headers/"
aliases = [
    "/post/shadowd_client_headers/",
]
author = "Hendrik Buchwald"
+++

It is now recommended to disable whitelist checks of client HTTP headers in Shadow Daemon to prevent false-positives.

<!--more-->

## Background

A HTTP client header is additional data that is passed to the web server by the client in a key-value format, for example self-identification as a specific browser through the user agent.
Historically the used headers were well defined and custom headers prefixed with `X-`.
This changed in 2012 when [RFC 6648 deprecated the `X-` prefix and similar constructs](https://tools.ietf.org/html/rfc6648).

Additionally, new headers are established by changes in the specification like [Cross-Origin Resource Sharing](https://developer.mozilla.org/de/docs/Web/HTTP/CORS), i.e. new clients may include new headers.
It is expected that the servers ignore headers that they do not know and need.

## Recommendation

The last years of running Shadow Daemon in production have shown that allowing all headers yields the best results in regards to false-positives/true-positives.
If you are running Shadow Daemon as a **web application firewall** do not restrict the length or character set of the headers!

<img width="100%" src="/img/shadowd_client_headers/whitelist.jpg" alt="Whitelist Config" />

If you are running Shadow Daemon as a **honeypot** you may want to consider to continue using a very narrow list of allowed headers.
Some malicious clients contain typos in header names and self-identify this way.
I recommend to have a look at the results yourself here: https://demo.shadowd.zecure.org/parameters?parameter_filter%5BincludePaths%5D%5B0%5D=SERVER%7CHTTP_%2A&parameter_filter%5BincludeNoWhitelistRule%5D=1

