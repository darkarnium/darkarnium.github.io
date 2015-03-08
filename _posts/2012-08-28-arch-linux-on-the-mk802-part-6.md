---
layout: post
title: Arch Linux on the MK802 (Part 6)
date: 2012-08-28 00:27:54.000000000 -07:00
author: Peter Adkins
image:
---
Although far from perfect, and a little bit - okay, a lot - verbose on the serial console, I've compiled and uploaded an ArchLinux MK802 image which is working on my MK802 1GB. It's worth noting that the image itself is only 2GB as it was built for a 2GB MicroSD card.

![ALARM running on the MK802](/assets/article_images/2012/mk802-alarm.png)

Note: This version does NOT have a frame-buffer working, nor X installed. If you do not have a serial console to your MK802 you will not be able to configure wireless, or get into the machine.

The uBoot, script.bin, Kernel and RootFS are all included in this image. It is a complete filesystem ready to be gunzipped, and then written out to a MicroSD card. As for versions:

* uBoot - Custom uBoot built from [uboot-allwinner](https://github.com/hno/uboot-allwinner) with the mods from [Here](https://github.com/hno/uboot-allwinner/issues/11)[^1].
* Kernel - Custom 3.0.39+ Kernel from [linux-allwinner](https://github.com/amery/linux-allwinner) sources
* script.bin - Taken from [lubuntu-desktop-12.04-4-720p-1GB-miniand.com.img](https://www.miniand.com/forums/forums/2/topics/1)
* Arch RootFS - Based on the Sun4i (Mele) RootFS from [ALARM](http://archlinuxarm.org/platforms/armv7/mele-a100)[^2].

You should be able to download the file, extract and confirm the SHA256 hash and then use dd to write directly to a MicroSD card. Once written, it should be bootable once inserted into an MK802 and powered on. The login details are the same as ALARM, which is 'root' for both the username and password - I'd seriously recommend changing this after you have first logged in.

The image can be found [ArchARM-3.0.39-MK802.img.gz](http://dl.dropbox.com/u/35852360/ArchARM-3.0.39-MK802.img.gz) and the SHA256 hash [ArchARM-3.0.39-MK802.img.sha256](http://dl.dropbox.com/u/35852360/ArchARM-3.0.39-MK802.img.sha256).

Again, this is a rough version so your mileage may vary.

[^1]: I built the uBoot from sources as I wanted to learn a bit more about the architecture and the pre-compiled uBoot - with the mods from GitHub linked to above - that I had downloaded from Miniand was not displaying the full 1GB of RAM on the system information at boot time. Although only cosmetic, as the full 1GB was available from the given operating system, I made some changes to have it both display and allocate / initialize the full 1GB.
[^2]: I have added a few packages to the ALARM version, namely wireless-tools and netcfg for wireless configuration - the latter of which has been added to the DAEMONS list. I have also added the RTL8192CU, MAC80211, MALI and HDMI kernel modules to be loaded on boot.
