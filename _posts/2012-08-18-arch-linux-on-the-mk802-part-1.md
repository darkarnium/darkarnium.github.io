---
layout: post
title: Arch Linux on the MK802 (Part 1)
date: 2012-08-18 22:18:59.000000000 -07:00
author: Peter Adkins
image:
---
Recently, I picked up an MK802 for about $100 including shipping. It's a pretty cool little unit, and the guys over at [Miniand](https://www.miniand.com/forums/forums/development) seem to be very helpful with breaking the unit open and hacking it to do as much as possible - which is really cool to see.

My goal is to use the unit for a small Linux I2C and servo interface to replace an Arduino which is doing just that. The advantage on the Linux platform is that I'm free to use Perl, Python, and C to do what needs to be done. The 802.11 interface may come in handy for uploading and downloading code, but an RS-232 interface is more than enough for the 'set and forget' configuration I'm looking at.

Due to the aforementioned reason I'm looking for a distribution which is truly lightweight, which has lead me to Arch Linux. Due to this - and there being an active ARM port - I've spent the last few days attempting to get it running from a 2GB μSD card.

However, first I needed to go back to the basics of embedded ARM systems, I'll cover a few caveats and tips I encountered in this series of posts.

In order to get a better understanding of the architecture, I've started with attempting to build my own u-boot image for the device. As the device is built on the AllWinner A10 system there is a decent amount of documentation floating around pertaining to the boot process and recompilation of u-boot images - one of the most helpful being the [uboot-allwinner GitHub Repository](https://github.com/hno/uboot-allwinner/wiki)

The first step in this process, is to get an RS-232 port to see exactly what is going on and to allow for recovery in the event of anything going drastically wrong. When opening the case and pulling out the main board you will very quickly notice 4 solder pads to the left of the A10 package. At first I was pinning these out manually with a multimeter, but it appears that someone else has already done it and [documented the pinout](https://www.miniand.com/forums/forums/2/topics/83).

![The result of 5 minutes with a soldering iron, a few beheaded jumper cables and a BusPirate](/assets/article_images/2012/mk802-buspirate.jpg)

Now that we can easily see the entire boot process, and interact with it, we can start delving deeper into the device and start compiling a custom u-boot image for it. Out of the box, for one reason or another, the u-boot only reports a total of 512MB of available memory. The main board has a pair of [Micron MT41J256M16-125](http://www.micron.com/parts/dram/ddr3-sdram/mt41j256m16re-125) ICs below the A10, which should allow for a maximum of 1Gigabyte of usable memory - minus the shared memory for the Mali GPU.

Let's see if we can fix this up first...