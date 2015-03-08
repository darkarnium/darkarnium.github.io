---
layout: post
title: Arch Linux On the MK802 (Part 2)
date: 2012-08-19 01:19:28.000000000 -07:00
author: Peter Adkins
image:
---
...all right, after a few hours I've finally managed to get a new U-Boot compiled with support for the full compliment of RAM installed on the MK802. This having been said, as to whether this has just been a cosmetic or functional change, I'm not one hundred percent certain.

A few bullet points as to how to start have been included below. For more detailed information on formatting the card and getting the Allwinner U-Boot sources, see [this great resource.](http://rhombus-tech.net/allwinner_a10/a10_mmc_boot/)

1. Check out the [Allwinner U-Boot](https://github.com/hno/uboot-allwinner/) sources via GIT.
2. Download and install install the [CodeSourcery ARM Toolchain](http://www.mentor.com/embedded-software/sourcery-tools/sourcery-codebench/editions/lite-edition/).
3. Compile a new U-Boot for the `sun4i` platform.
4. Format a new μSD Card with a FAT16 and EXT3 partition - the former starting from block 2048.
5. Use `dd` to install the newly compiled SPL and U-Boot image onto the μSD card.
6. Insert the μSD card into the MK802 and boot.

![uBoot Memory](/assets/article_images/2012/mk802-uboot-memory.png)

Again, this is to be taken with a grain of salt for the moment as I have not been able to confirm whether this has indeed allowed for more than 512MB of memory to be addressed. We should know more once I have a Linux Kernel running on the board.

Still more to come.

*Edit:* This turned out to be a cosmetic change, any Kernel booted would only detect a 512MB of available memory. There is some information floating around on how a patched 1GB U-Boot was prepared over at the hno-uboot project's [Issue Page](https://github.com/hno/uboot-allwinner/issues/11).