---
layout: post
title: Raspberry Pi, Python and I2C (Part 1)
date: 2013-01-11 20:39:23.000000000 -08:00
author: Peter Adkins
image:
---
As part of a recent project, I have had to take a crash course in I2C in order to interface with a number of sensors - namely an accelerometer and gyroscope. I've included some of the issues I came across in this post, as it may assist someone else out there :)

The platform I have chosen for the work is a Raspberry Pi. I had previously build a shield for an Arduino Duemilanove - with appropriate TTL level-shifters - but I've since opted for the Raspberry Pi as it allows for a greater amount of flexibility required for the project.

In order to connect the sensors to the Pi - assuming they are bother 3.3v powered and 3.3v logic - we will need the following pins on the [GPIO header](http://elinux.org/RPi_Low-level_peripherals#General_Purpose_Input.2FOutput_.28GPIO.29):

* Pin 1 - 3.3V
* Pin 3 - I2C SDA (Data Line)
* Pin 5 - I2C SCL (Clock Line)
* Pin 6 - Ground

The connection to your sensors will vary. However, at least in the case of the [BMA180](https://www.sparkfun.com/products/9723) and [ITG3200](https://www.sparkfun.com/products/9801) I am using, connection was relatively simple; just connect the 'dots' - so to speak, and you should be fine [^1]

Once your sensors are wired up and connected to your Pi, you can power it on and start getting into the software side of things. Depending on the OS running on your Pi, additional work may be required to enable I2C. In my case - running Arch Linux - it was as simple as loading the `i2c_dev` kernel module with `modprobe i2c_dev`.

Once the module has been loaded, you can verify whether you are able to communicate with your I2C using the 'i2cdetect' and 'i2cdump' binaries - which are part of the package `i2c-tools` under Arch Linux.

First off, we'll get a listing of available I2C buses on the system with `i2cdetect -l`. You should receive something like the following in response to this query. If you do not, double check that the `i2c-dev` and `bcm2708_i2c` kernel modules are loaded:

{% highlight bash %}
i2c-0   i2c             bcm2708_i2c.0                           I2C adapter
i2c-1   i2c             bcm2708_i2c.1                           I2C adapter
{% endhighlight %}

All right, the system is showing that there are two available I2C buses on the system. However, we'll need to find exactly which bus our sensors are connected to; this is where 'i2cdetect' really comes in handy.

In order for slave devices to communicate on an I2C bus, they have an address assigned. These addresses should be able to be be found in the datasheet(s) for your given device / sensor, and may even be configurable. However, please check the data sheets for your devices if you are attaching multiple devices to the same bus, as if the addresses are the same, you may encounter address conflicts. Think of it as if walking into a room and addressing an audience by a single name, if there are two people listening with the same name you are going to receive the attention of both.

In order to list the devices connected to your I2C bus, use `i2cdetect <BUSNUMBER>`. As an example, I've included the output of the above command on the second I2C bus on the Pi - with sensors attached:

{% highlight bash %}
[user@host bin]# i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- 69 -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
{% endhighlight %}

As you can see from the above, there are two devices communicating on the bus, with addresses of 0x40 (BMA180) and 0x69 (ITG3200).
[^1]: There may be additional pins that need to be connected on your sensor, check the data sheet(s) for more information.