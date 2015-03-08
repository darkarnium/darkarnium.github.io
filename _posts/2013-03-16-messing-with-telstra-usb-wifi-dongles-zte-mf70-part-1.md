---
layout: post
title: Messing with Telstra USB + WIFI Dongles (ZTE MF70) (Part 1)
date: 2013-03-16 02:29:15.000000000 -07:00
author: Peter Adkins
image: 
---
Earlier today I made a trip down to a local electronics store looking for a low cost Telstra 3G dongle I could pull apart for use in a project. I found exactly what I was looking for in a AUD$25.00 Telstra E353T dongle, however, much to my dismay, they were out of stock. The only item they had around that price point - well, slightly more - was a Telstra 3G USB + WiFi dongle. In the end, I decided that I'd grab one as the built in 802.11 access point screamed embedded Linux. As it turns out, this device is actually a re-branded ZTE MF70.

As soon as I got the device home, the first thing I did was liberate it from it's plastic enclosure, and start hunting for a serial interface. I found two potential candidates, one was three solder pads next to each other - one of which was a ground - and the other was 14 solder pads aligned in two rows of seven. I ran a multi-meter over the pads looking for what appeared to be a serial console to no avail. As a last ditch attempt to get a serial console, I soldered onto the three pads, and swapped the potential RX / TX lines and baud rates until I found what I was looking for; unfortunately, this didn't yield any garbage or even a console.

So, next stop, see whether there was any telnet or SSH services running on the device. To my surprise, I found that there did appear to be a telnet service running, but was being reported as 'filtered.' So, I whacked together a quick bash script to iterate through all possible IPs in the MF70's subnet and check whether we could get a response - to determine whether the manufacturer had firewalled the telnet service to a single IP, or set of IPs. Alas, once again, this didn't yield any positive results.

So, next up, I went over the management interfaces, checking to see whether there was any diagnostic tools built in we might be able to use to our advantage. Much to my surprise, it appeared that the HTTP management interface had been sanitized quite well... Except for the following.

![The Webshare UI](/assets/article_images/2013/mf70-webshare.png)

This is web share. It's supposed to allow you to access files from an inserted μSD via the MF70s web interface, but instead it also gives you access to the root filesystem with a bit of HTTP tampering. It appears that the only 'security' here is implemented in javascript. Please ignore the files in the listing above, the μSD card I inserted was from an old phone so it had data from an old Android install on it. As such the previous image does not show you the root file system, just the '/mmc2' directory.

However, all we needed to do was tamper with the HTTP post variables and change the `path_SD_CARD=%2Fmmc2` variable to `path_SD_CARD=%2F`

![The MF70 root directory](/assets/article_images/2013/mf70-webshare_root.png)

...and say hello to the root file system of the device. We attempted to use the 'download' links to grab configuration files for further inspection, but unfortunately, this wouldn't work unless the files are already inside the web directory, and even then, file were served by the web server itself via normal methods - ending in any ASP scripts being executed and their output returned. So, we decided to go hunting for interesting files inside the web directory instead. This is where we found something interesting.

If you browse the the following URL when connected to this particular device, you are presented with a plain text configuration file including passwords, WPA Keys and SSIDs, as well as APN information etc. To make things even better, no authentication is required. Now, just imagine the 'Remote Management (via WAN)' feature was enabled. This is a bit terrible, especially considering what you can do with this information:

[http://192.168.0.1/userseting_nvconfig.txt](http://192.168.0.1/userseting_nvconfig.txt) or [http://m.home/userseting_nvconfig.txt](http://m.home/userseting_nvconfig.txt)

More to come...