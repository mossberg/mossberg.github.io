---
layout: post
date: 2014-01-10 17:30
title: Banner hiding on Ubuntu 12.04
categories: security reference linux sysadmin
---

In most cases, the first phase of any sort of cyber-attack is the "fingerprinting" phase. This essentially involves an attacker attempting to ascertain as much information as possible about the target in question, say, a web server also running an ssh service. Some basic pieces of info that the attacker would be interesting in gathering would be the OS installed on the server, and the types of web and ssh servers running as well as version numbers for all of these. With this info, at the very least, the attacker would be able to google *"[web/ssh server] [version] vulnerabilities"*, so you can see how it might be a good idea to keep this sort of info hidden. Of course, there is a valid [security through obscurity](http://en.wikipedia.org/wiki/Security_through_obscurity) argument to be made here, and even the Apache docs[^1] themselves state that 

> ...disabling the Server: header does nothing at all to make your server more secure; the idea of "security through obscurity" is a myth and leads to a false sense of safety. 

While I can see how it's not acceptable to use obscurity as a complete solution for potential vulnerabilities, I disagree with the assertion that it does "nothing at all." Obviously, everyone should aim to make their systems secure at a fundamental level, and obscurity is no substitute for that, however every little bit that you can do to make it more challenging for an attacker, in order to bolster *your existing security implementation*, is worth it, in my opinion.

Moving back to the original topic, you might be surprised to know how much of this information (services, version numbers) your server gives away by default. For example, let's take a look at the response headers from a default Apache install:

[![Screen_Shot_2013-12-15_at_1.38.13_AM.png](https://d23f6h5jpj26xu.cloudfront.net/sgoa4yt9b3putw_small.png)](http://img.svbtle.com/sgoa4yt9b3putw.png)

Right off the bat, we can see that we're giving away our OS, web server, and version. Let's make it a little harder for an attacker and hide everything except for "Apache", which can be done with a trivial edit to your `/etc/apache2/apache2.conf` file. Simply append the following lines at the end of the file.

    ServerTokens Prod
    ServerSignature Off

Technically all you really need is the first line, which will do what we want, but the second line will also remove server info in Apache generated pages (error pages, directory listings), so it's probably a good idea too. That gets us this far:

[![Screen_Shot_2013-12-15_at_2.01.18_AM.png](https://d23f6h5jpj26xu.cloudfront.net/1gs6rgbxmeliq_small.png)](http://img.svbtle.com/1gs6rgbxmeliq.png)

Unfortunately, "Apache" is as far as we can go with these configuration files; there is no option to turn the server header completely off. However, there is one other thing we can do... get out your hex editors[^2], boys, we're going binary patching! Yes, you heard me right, we're going to directly edit the apache2 binary and remove that pesky banner string. 

First off, I recommend making a copy of the binary so if you really screw something up, you can easily get back to where you were. Then get out hexedit and search in the ASCII section for the string "Apache". Eventually you'll see some familiar strings all in a row, with the Apache version numbers, etc. There are a couple of them; they represent the different options with the "ServerTokens" config. The "Prod" option corresponds to the first instance of "Apache" there, so let's change it to "LOLche". Make sure to only overwrite the existing bytes and nothing more, because doing otherwise could cause other problems with the binary.

[![Screen_Shot_2013-12-15_at_2.17.43_AM.png](https://d23f6h5jpj26xu.cloudfront.net/tok3zivx32pngq_small.png)](http://img.svbtle.com/tok3zivx32pngq.png)

Restart the apache service, and violá! We're officially running a "LOLche" web server.

[![Screen Shot 2013-12-15 at 2.24.33 AM.png](https://d23f6h5jpj26xu.cloudfront.net/rhhj4sha4u4jug_small.png)](http://img.svbtle.com/rhhj4sha4u4jug.png)

While we're messing with these headers, I should point out that you can also add your own headers too. Make sure you have the mod_headers.c Apache module enabled, and then simply append the following to your `/etc/apache2/httpd.conf`.


    <IfModule mod_headers.c>
        Header set Servar "y u look here go away"
    </IfModule>

Which gives you:

[![Screen_Shot_2013-12-15_at_2.30.28_AM.png](https://d23f6h5jpj26xu.cloudfront.net/9lml4ymw40acog_small.png)](http://img.svbtle.com/9lml4ymw40acog.png)

Similar to Apache, OpenSSH also gives out a bunch of information by default.

    $ nc localhost 22
    SSH-2.0-OpenSSH_5.9p1 Debian-5ubuntu1

To get rid of the Debian information, just add this line to your `/etc/ssh/sshd_config`.

    DebianBanner no

However, this still leaves all the OpenSSH stuff. A more complete solution would be to patch the sshd binary in the same way we did the apache binary.

In addition to screwing with people that are physically looking at these headers and stuff, making these quick fixes also makes an nmap port scan much less effective, since this is primarily where nmap gets its information about what services you're running.

Just to reiterate, these configurations are *not* designed to be security solutions in and of themselves, but are simply meant to bolster your existing security practices and make it just a little harder for attackers to do their thing.

[^1]: http://httpd.apache.org/docs/current/mod/core.html

[^2]: I guess you could also use a plain text editor just as well also.
