---
layout: post
title: Urwid on a Crystalfontz 635 LCD
categories: [Python]
redirect_from: /article/2010/03/urwid-crystalfontz-635-lcd/
excerpt_separator: <!--more-->
---

![XTerm 256-Colour Chart (cows)](/images/cflcd.jpg)

The development version of Urwid now has support for display and input on a Crystalfontz 635 LCD panel. These are small LCD screens that fit in a PC's 5¼" drive bay. They have six buttons, four red/green LEDs and a 20x4 character display with an adjustable backlight.

<!--more-->

This sort of device is great for simple interfaces to headless machines, like media PCs and special-purpose systems. Urwid simplifies building an interface by taking care of text wrapping, scrolling and cursor movement. The new `lcd_display.py` module also handles CRCs, acknowledgements and can simulate key repeating so the device acts more like a regular keyboard. It should also be easy to add support for other similar devices.

The new example program `lcd_cf635.py` demonstrates the new display module and all of the features of this device. It also provides a good example of how to build simple menus and customize widgets for a device like this.
