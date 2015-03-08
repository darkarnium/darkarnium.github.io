---
layout: post
title: Arch Linux on the MK802 (Part 5)
date: 2012-08-24 22:14:20.000000000 -07:00
author: Peter Adkins
image:
---
All right, after a bit of fiddling, I've managed to get Arch ARM running on the MK802 well enough to automatically boot straight into Arch ARM and bring up the 802.11 interface. I'm currently attempting to get the frame-buffer working, as the only interface available is console via RS232 - or SSH once the machine is connected to an access-point.

{% highlight bash %}
[root@sun4i ~]# lsmod
Module                  Size  Used by
mali                  138158  0
hdmi                   15360  0
disp                  229420  1 hdmi
ump                    46739  2 disp,mali
8192cu                521118  0
mac80211              183378  0
{% endhighlight %}
{% highlight bash %}
[root@sun4i log]# ifconfig wlan0
wlan0: flags=4098<BROADCAST,MULTICAST>  mtu 1500  metric 1
        ether 48:02:2a:XX:XX:XX  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes   0 (0.0 B)
        RX errors  0  dropped 0 overruns 0  frame   0
        TX packets 0  bytes   0 (0.0 B)
        TX errors  0  dropped 0 overruns 0  carrier 0  collisions 0
{% endhighlight %}

The system itself is currently running from a 2GB MicroSD card, with 1.2GB free - as the install is only 437MB in size. However, the boot up is still quite verbose and "dirty" so I need to ensure that everything is healthy and modules not required are stripped from the kernel.

I'm still doing some testing to ensure things are working at a satisfactory level, once I'm relatively happy with the state of things I'll compile an image file to DD straight over an SD card ready to boot - including uBoot. I'll also upload my Kernel config and uBoot amendments - based on [this discussion](https://github.com/hno/uboot-allwinner/issues/11).

Happy hacking :)
