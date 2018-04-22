+++
title = "DIY Raspberry Pi Alarm System"
description = ""
tags = [
    "release",
    "alarm",
    "diy",
    "raspberry pi"
]
date = "2018-04-22"
categories = [
    "Development"
]
menu = "main"
+++

In this blog post I will explain how to turn a **Raspberry Pi** into an **alarm system** that **detects movements**, **records** them, and **sends** the images to your **phone**.
Additionally, the alarm system will turn on and off automatically based on the location of your phone.

<!--more-->

## Motivation
There is a certain comfort in knowing that your home is safe and secure. This can be easily achieved by installing an alarm system. There are more than enough vendors out there that sell their systems for a reasonable price, they have a few major drawbacks though:
Most systems are closed, connected to a cloud that you have no control over, and - as history as shown often enough - they tend to be incredibly insecure.
So basically they are a blackbox bug that you install in your own home. Bummer.

This was not an option for me, so I decided to make my own alarm system instead consisting only out of [free software](https://en.wikipedia.org/wiki/Free_software). How hard could it possibly be?
Spoiler, not very. It still took some time and effort to create and combine all required components as there was no existing software that met all my requirements.

This blog post documents the setup process of my system.

## Requirements
The requirements are very simple. All you need is a Linux system with a web cam and a Linux server.
If you want to be able to copy all configuration files from this post though you should make sure to use the same hard- and software as I do.

My alarm node is build out of a **Raspberry Pi 3** running **Raspbian Stretch**. Any Raspberry Pi model will do but model 3 has built-in wifi which is very handy.

TODO: image + link

As camera I use the **Electreeks Raspberry Pi Kamera Modul**. It is equipped with infrared lights (optional) and works both at day and at night. It is also rather small compared to a normal web cam.
But almost any other camera module or USB web cam will do as well.

TODO: image + link

My server is running **Ubuntu 16.04**. It does not require much ressources, so any cheap VPS will do.
Keep in mind though that this server controls the alarm system, so pick a trustworthy hoster or host it yourself.
You will need a (sub-) domain for TLS encryption.

TODO: image + link

## Server Setup
The server is the central manager of the alarm system. It stores the captured images, sends them to the admins via e-mail, and it tells the local alarm systems if they should turn on or off.
The server is a Symfony 4 web application that provides a small HTTP API as well as a small web interface.

We start by installing all required tools on our Linux server.

    apt update
    apt install lighttpd php-cgi php-cli composer git motion software-properties-common
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install certbot

The next step is to download the *alarm server* and to initialize a *SQLite* database.

    cd /var/www
    git clone https://github.com/zecure/alarm-server
    cd alarm-server
    composer install
    ./bin/console doctrine:database:create
    ./bin/console doctrine:schema:update --force


Next you have to create user accounts.

Your main account has the role *admin*. You can use this account to access the alarm server with a browser.
All admin accounts are informed by e-mail about new alarms and dead clients. Use the e-mail address of your phone to receive the alarms there.

If you want to disable or enable the alarm automatically based on events you need to have an account with the role *api* as well.
For security reasons this has to be a different user than the admin user because the API is not protected by an anti-CSRF token.

Your third account has the role *alarm*. Create an own user for every *Raspberry Pi* that you want to deploy.

    ./bin/console fos:user:create admin
    ./bin/console fos:user:promote admin ROLE_ADMIN
    ./bin/console fos:user:create api
    ./bin/console fos:user:promote api ROLE_API
    ./bin/console fos:user:create alarm
    ./bin/console fos:user:promote alarm ROLE_ALARM

Now all that is left to do is to configure our web server, *lighttpd*. To securely start the web server we need a valid TLS certificate though.
It can be obtained from LetsEncrypt.

    lighty-enable-mod fastcgi fastcgi-php
    cd /etc/lighttpd/
    openssl dhparam -out dhparam.pem 4096

Create the file `/usr/bin/letsencrypt_renew` with the following content. Make sure to set `domain` and `email` correctly.

    #!/usr/bin/env bash
    set -e

    # begin configuration
    domain="foo.bar.org"
    email=foo@bar.org
    w_root=/var/www/html/
    user=www-data
    group=www-data
    # end configuration

    if [ "$EUID" -ne 0 ]; then
        echo  "Please run as root"
        exit 1
    fi

    /usr/bin/certbot certonly --agree-tos --renew-by-default --email $email --webroot -w $w_root -d ${domain}
    cat /etc/letsencrypt/live/$domain/privkey.pem  /etc/letsencrypt/live/$domain/cert.pem > ssl.pem
    cp ssl.pem /etc/lighttpd/letsencrypt.pem
    cp /etc/letsencrypt/live/$domain/fullchain.pem /etc/lighttpd/
    chown -R $user:$group /etc/lighttpd/
    rm ssl.pem

    /etc/init.d/lighttpd restart

The file has to be executable and it has to be executed periodically to refresh our certificate.

    chmod 755 /usr/bin/letsencrypt_renew
    ln -s /usr/bin/letsencrypt_renew /etc/cron.monthly/letsencrypt
    /usr/bin/letsencrypt_renew

Next, replace `/etc/lighttpd/lighttpd.conf` with the following content.

    server.modules = (
	    "mod_access",
	    "mod_alias",
	    "mod_compress",
	    "mod_redirect",
	    "mod_rewrite",
    )

    server.document-root        = "/var/www/html"
    server.upload-dirs          = ( "/var/cache/lighttpd/uploads" )
    server.errorlog             = "/var/log/lighttpd/error.log"
    server.pid-file             = "/var/run/lighttpd.pid"
    server.username             = "www-data"
    server.groupname            = "www-data"
    server.port                 = 80
    server.reject-expect-100-with-417 = "disable"

    index-file.names            = ( "index.php", "index.html", "index.lighttpd.html" )
    url.access-deny             = ( "~", ".inc" )
    static-file.exclude-extensions = ( ".php", ".pl", ".fcgi" )

    compress.cache-dir          = "/var/cache/lighttpd/compress/"
    compress.filetype           = ( "application/javascript", "text/css", "text/html", "text/plain" )

    include_shell "/usr/share/lighttpd/create-mime.assign.pl"
    include_shell "/usr/share/lighttpd/include-conf-enabled.pl"

    $HTTP["scheme"] == "http" {
	    $HTTP["url"] !~ "^/.well-known/" {
		    $HTTP["host"] =~ ".*" {
			    url.redirect = (".*" => "https://%0$0")
		    }
	    }
    }

Time to create the initial certificate by running:

    /usr/bin/letsencrypt_renew

Now that the certificate exists we can append the following content to `/etc/lighttpd/lighttpd.conf` to enable HTTPS access to our alarm server.

    $SERVER["socket"] == ":443" {
	    ssl.engine = "enable"
	    ssl.ca-file = "/etc/lighttpd/fullchain.pem"
	    ssl.pemfile = "/etc/lighttpd/letsencrypt.pem"
	    ssl.dh-file = "/etc/lighttpd/dhparam.pem"

	    #
	    # Mitigate BEAST attack:
	    #
	    # A stricter base cipher suite. For details see:
	    # http://blog.ivanristic.com/2011/10/mitigating-the-beast-attack-on-tls.html
	    #
	    ssl.cipher-list = "ECDHE-RSA-CHACHA20-POLY1305 ECDHE-ECDSA-CHACHA20-POLY1305 AES128+EECDH:AES128+EDH:!aNULL:!eNULL"
	    #
	    # Make the server prefer the order of the server side cipher suite instead of the client suite.
	    # This is necessary to mitigate the BEAST attack (unless you disable all non RC4 algorithms).
	    # This option is enabled by default, but only used if ssl.cipher-list is set.
	    #
	    ssl.honor-cipher-order = "enable"
	    #
	    # Mitigate CVE-2009-3555 by disabling client triggered renegotation
	    # This is enabled by default.
	    #
	    ssl.disable-client-renegotiation = "enable"
	    #
	    # Disable SSLv2 because is insecure
	    ssl.use-sslv2 = "disable"
	    #
	    # Disable SSLv3 (can break compatibility with some old browser) /cares
	    ssl.use-sslv3 = "disable"

	    server.document-root = "/var/www/alarm-server/public"
	    url.rewrite-if-not-file = ( "(.+)" => "/index.php$1" )
    }

Don't forget to restart *lighttpd* to load the new configuration file.

    /etc/init.d/lighttpd restart


## Client Setup

## Phone Control
