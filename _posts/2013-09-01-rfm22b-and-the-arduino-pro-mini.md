---
layout: post
title: RFM22B and the Arduino Pro Mini
date: 2013-09-01 21:51:28.000000000 -07:00
author: Peter Adkins
image: 
---
As part of new project to monitor a few systems around the house I have been looking for a stable way to collect and transmit data to a central location as part of a sensor network. This project lead me to the RFM22B, after a recommendation from a colleague around the RFM12, and the Arudino Pro Mini.

Previously I had a system which was based on a Raspberry Pi with an 802.11 dongle in one of the USB ports and a custom GPIO expansion board providing I2C and 1-Wire interfaces. However, running a full linux stack for what is effectively shuffling a few bits about seemed a bit too much overhead. Not to mention, double brick walls and water pipes tend to wreak havoc with 802.11 - of which my house has a few, and all of which are in very inconvenient locations.

After a bit of searching, and after some discussions with a colleague, an Arduino Pro 'strapped' to an RFM22B appeared to be the best suited tool for this task.

What was great about this configuration - compared to the Raspberry Pi - is that the Arudino Pro could provide ADC for various analog sensors, as well as providing SPI, I2C and 1-Wire without too much hassle and with a very small power footprint. As a result, it could even be strapped to a set of low-cost AA / AAA batteries and run without the need to supply a wall-wart to provide 5V.

In order to make things easier in the long run, I've put together a simple PCB to breakout the available pins from the Arduino Pro - while still connecting the required SPI pins to the RFM module - as well as providing a simple 1-Wire interface separate from the analog and digital IO pins.

The boards in the photograph above is the first run of this breakout. 

Unfortunately, in my ultimate wisdom, I managed to move my top-silkscreen labels onto another layer, which I didn't notice until the Gerbers had been accepted by the fab. As a result there was no proper silkscreen, and the labels that were left were all printed as gibberish. This was entirely my fault, not the fab. Luckily, this was only a cosmetic issue, but it certainly was annoying.

All in all, the board provides a pad for a whip antenna for the RFM22B, eight digital I/O pins, six analog I/O pins, a 1-wire bus - with external pull-up resistor - a three pin UART header for programming, and VIN - which has been connected to the RAW input on the Arudino Pro to allow for battery power.

I'm still waiting on some final parts to arrive before I can confirm that this board is working, at such a time I will share the eagle schematics and board layout files. Although this is a VERY simple board and can be knocked out in about 10 minutes in eagle, it might save someone out there some time :)