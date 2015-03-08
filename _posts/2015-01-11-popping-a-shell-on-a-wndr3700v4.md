---
layout: post
title: Popping a shell on a WNDR3700v4
date: 2015-01-11 16:44:16.000000000 -08:00
author: Peter Adkins
image: 
---
I recently received a WNDR3700v4 for use with my cable connection. My first question regarding the device was whether it is possible to pop a shell on the underlying OpenWRT system without cracking open the case and connecting to a serial console.

To start with, I grabbed a copy of the latest firmware from the NetGear website and run it through binwalk. As expected, a squashfs filesystem and an "unknown" binary file fell out - which looks like the kernel.

The squashfs filesystem required me to download and compile the GPL package for this router, and compile the 'squashfs-tools' from the sources distributed by NetGear. Once this was done, I was able to unpack the squashfs filesystem without issue. After this had unpacked, I rifled through the GPL package for any binaries distributed - which are usually a good location to look for custom glue / logic. A few files stood out, but the most interesting was `net-cgi`.

After a couple of hours of digging through the `net-cgi` application for this device, I managed to find a diagnostic page which allows arbitrary command injection - albeit authenticated. This appears to be as a result of the input from the user being passed via `system()` call directly as a string ("%s"). A quick Google shows that someone had already found this page back in 2013, published it and received a CVE yet it still hasn't yet been removed from the NetGear firmware.

Interestingly enough, it appears that this page is attempting to perform at least some input sanitization, as both commands wrapped inside of `$()` were escaped, as were commands wrapped in backticks. However, the most basic injection / follow-on method - being to simply pre-pend a semicolon - was not filtered.

The page can be found at the following URL - assuming you are using the default NetGear dnsmasq configuration - [http://www.routerlogin.net/ping6\_traceroute6\_hidden\_info.htm](http://www.routerlogin.net/ping6_traceroute6_hidden_info.htm)

The shell can be spawned through the use of the following command: `; telnetd -p 1337 -l /bin/login`

The net result was an unauthenticated root-shell on TCP 1337 (as TCP 23 is already in use with a seemingly unresponsive telnetd instance):

![Ready to do your bidding](/assets/article_images/2015/wndr-shell.png)

Not bad.