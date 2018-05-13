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
url = "/diy-home-alarm-system/"
aliases = [
    "/post/diy-home-alarm-system/",
]
author = "Hendrik Buchwald"
+++

In this blog post I will explain how to turn a **Raspberry Pi** into a **home alarm system** that **detects movements**, **records** them, and **sends** the images to your **phone**.
Additionally, the alarm system will turn on and off automatically based on the location of your phone.

<img width="49%" src="/img/alarm/demo2.jpg" />
<img width="49%" src="/img/alarm/demo1.jpg" />

<!--more-->

## Motivation
There is a certain comfort in knowing that your home is safe and secure. This can be easily achieved by installing an alarm system. There are more than enough vendors out there that sell their systems for a reasonable price, they have a few major drawbacks though:
Most systems are closed, connected to a cloud that you have no control over, and - as history has shown often enough - they tend to be incredibly insecure.
So basically they are a blackbox bug that you install in your own home. Bummer.

This was not an option for me, so I decided to make my own alarm system instead consisting out of [free software](https://en.wikipedia.org/wiki/Free_software). How hard could it possibly be?
Spoiler, not very. It still took some time and effort to create and combine all required components as there was no existing software that met all my requirements, so I had to develop my own.

This blog post documents the whole setup process, both as a documentation for myself and as a tutorial for other people that also want to build their own secure alarm system.

## Requirements
The requirements are very simple. All you need is a Linux system with a camera, a Linux server, and some Linux experience.
If you want to be able to copy all configuration files from this post though you should make sure to use the same hard- and software as I do.

My alarm system is build with a **Raspberry Pi 3** running **Raspbian Stretch**. Any Raspberry Pi model will do but model 3 has built-in wifi which is very handy.

<a href="https://www.raspberrypi.org/products/raspberry-pi-3-model-b/" target="_blank">
  <img width="60%" src="/img/alarm/raspberry-pi.jpg" alt="Raspberry Pi 3" />
</a>

As camera I use the **Electreeks Raspberry Pi Kamera Modul**. It is equipped with infrared lights (optional) and works both at day and at night. It is also rather small compared to a normal web cam.
But almost any other camera module or USB web cam will do as well.

<a href="http://electreeks.de/produkte/" target="_blank">
  <img width="60%" src="/img/alarm/electreeks.png" alt="Electreeks Raspberry Pi Kamera Modul" />
</a>

My server is running **Ubuntu 16.04**. It does not require many resources, so any cheap VPS will do.
Keep in mind though that this server controls the alarm system, so pick a trustworthy hoster or host it yourself.
You will need a (sub-) domain for TLS encryption and there should be no other major applications running on the same host.

## Server Setup
The server is the manager of the alarm system. It stores the captured images, sends them to the admins via e-mail, and it tells the local alarm systems if they should turn on or off.
The server is a Symfony 4 web application that provides a small HTTP API as well as a small web interface.

We start by installing all required tools on our Linux server.

    add-apt-repository ppa:certbot/certbot
    apt update
    apt install lighttpd php-cgi php-cli php-sqlite3 php-mbstring php-xml php-zip composer git certbot

The next step is to download the [alarm server](https://github.com/zecure/alarm-server) and to initialize a *SQLite* database.

    cd /var/www
    git clone https://github.com/zecure/alarm-server
    cd alarm-server
    composer install
    ./bin/console doctrine:database:create
    ./bin/console doctrine:schema:update --force
    chown -R www-data:www-data var/

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

    index-file.names            = ( "index.php", "index.html" )
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

Before the web application can be used some configuration parameters have to be set. This can be done in the file `/etc/lighttpd/conf-enabled/15-fastcgi-php.conf`.
Make sure to set `APP_SECRET`, `MAILER_FROM`, and `MAILER_URL` correctly.

    fastcgi.server += ( ".php" => 
	    ((
		    "bin-path" => "/usr/bin/php-cgi",
		    "socket" => "/var/run/lighttpd/php.socket",
		    "max-procs" => 1,
		    "bin-environment" => ( 
			    "PHP_FCGI_CHILDREN" => "4",
			    "PHP_FCGI_MAX_REQUESTS" => "10000",
			    "APP_ENV" => "prod",
			    "APP_SECRET" => "replace with random value",
			    "MAILER_FROM" => "alarm@example.org",
			    "DATABASE_URL" => "sqlite:///%kernel.project_dir%/var/data/data.sqlite",
			    "MAILER_URL" => "smtp://user:password@example.org:587?encryption=tls"
		    ),
		    "bin-copy-environment" => (
			    "PATH", "SHELL", "USER"
		    ),
		    "broken-scriptfilename" => "enable"
	    ))
    )

Don't forget to restart *lighttpd* to load the new configuration changes.

    /etc/init.d/lighttpd restart


## Client Setup
Prepare a *Raspberry Pi* with *Raspbian Stretch*, install the camera, and connect it to the internet.

If you are using a Raspberry Pi camera module you have to enable it first in the *Raspberry Pi Configuration*.

<img width="100%" src="/img/alarm/raspberry-pi_desktop.jpg" alt="Raspberry Pi Desktop" />

Navigate to *Preferences*, *Raspberry Pi Configuration* as shown in the image above.

<img width="49%" src="/img/alarm/raspberry-pi_config2.png" />
<img width="49%" src="/img/alarm/raspberry-pi_config1.png" />

There you can enable the camera in the *Interfaces* tab.
It is also highly recommended to **change the password**, **disable the auto login**, and to **boot to cli** in the *System* tab.

If you are using a Raspberry Pi camera module you also have to load the kernel module `bcm2835-v4l2` to create a `/dev/video` device for the camera.
A video device is required for `motion` to work properly. You can load the module manually by executing the following.

    modprobe bcm2835-v4l2

Since you do not want load it manually every time you start your Raspberry Pi add `bcm2835-v4l2` to the file `/etc/modules` instead.

Next, install the required software.

    apt update
    apt install motion

Replace the file `/etc/motion/motion.conf` with the following content.

    daemon on
    process_id_file /var/lib/motion/motion.pid
    videodevice /dev/video0
    
    width 800
    height 600
    framerate 5
    minimum_frame_time 1
    lightswitch 50
    threshold 2500
    
    output_pictures on
    target_dir /var/lib/motion/images
    on_picture_save curl -F "alarm[file]=@%f" -u alarm:password https://example.org/upload

This tool will automatically record images if it detects motion. Every time this happens `curl` gets called and it uploads the image to our server.
Make sure to replace the credentials and server address for `on_picture_save`. You might want to change other parameters as well in case you are not happy with the results of `motion`.

Finally, create the file `/usr/bin/alarm_status`, make it executable, and run it every minute with cron.

    #!/bin/bash
    curl --fail -u alarm:password https://example.org/ping
    if [ $? -eq 0 ]; then
       /etc/init.d/motion start
    else
       /etc/init.d/motion stop
       killall motion
       rm /var/lib/motion/motion.pid
    fi

Make sure to replace the credentials and server address for `curl`.

    chmod 750 /usr/bin/alarm_status
    crontab -e

    * * * * * /usr/bin/alarm_status

This will both tell the server that the Raspberry Pi is still alive and it also allows to turn `motion` on or off without having to expose ports.

You may also want to automatically delete old images from the Raspberry Pi after a certain period of time to avoid having to interact with the Raspberry Pi manually.
It can easily be done by creating the file `/etc/cron.hourly/motionrm` with the following content.

    #!/bin/sh
    find /var/lib/motion/images/ -type f -mmin +120 -exec rm {} \;

Do not forget to make the file executable by running `chmod +x /etc/cron.hourly/motionrm`.

## Phone Control
I do not want to enable or disable the alarm system manually, that would be too much work.
My first approach consisted of a script that tried to detect my mobile phone in the network.
Unfortunately the phone automatically enters a power saving mode from time to time where it will not be reachable in the network, so this approach did not work as expected.
So I moved to the Android application *Tasker* instead that allows me to execute actions on certain events. It is not free software but it does its job well.
Currently, I am using the presence of my wifi to enable or disable the alarm.

<img width="49%" src="/img/alarm/tasker2.png" />
<img width="49%" src="/img/alarm/tasker3.png" />

Alternatively you could also enable or disable motion with cron jobs at certain times to avoid having to use *Tasker*.
It is also possible to enable or disable the alarm manually through the web interface as well. Simply use your admin user to access `/status`.

<img width="100%" src="/img/alarm/tasker1.png" />

And that is it. You now have a fully automated alarm system that stores the recordings safely, informs you on your phone, and it is easily extendable with more cameras.

If you have questions or improvements feel free to post them [here](https://github.com/zecure/blog.zecure.org/issues).
