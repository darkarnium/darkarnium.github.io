---
layout: post
title: Arch Linux On the MK802 (Part 3)
date: 2012-08-19 11:07:14.000000000 -07:00
author: Peter Adkins
image:
---
While reading over the Arch Arm forums and developer documents I came across a few posts regarding the MK802. There is some suggestion that the Mele A1000 Arch build should work with the MK802 with a few modifications.

As a test, I've attempted to boot the sun4i builds from the [Arch Arm mirrors](http://us.mirror.archlinuxarm.org/os/) without much success. Unfortunately, a number of errors are thrown on boot before an NPE, Kernel Oops then subsequent kernel panic:

![Oops](/assets/article_images/2012/mk802-oops.png)

I haven't finished going over the exceptions just yet, but it looks like we might need a new Kernel; however, there is a chance that the issue is with the script.bin or uEnv. The silver lining here is that we can most probably use the RootFS from the M1000 distribution and just worry about getting the kernel running on the board, which should cut down the time to get a working Arch system by some time.

More to come...
