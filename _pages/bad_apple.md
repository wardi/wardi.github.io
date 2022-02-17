---
layout: page
title: Bad Apple on 32K EEPROM
permalink: /bad-apple/
---

[![Bad Apple on 32K EEPROM](/images/bad-apple.jpg)
Watch "Bad Apple on 32K EEPROM" on Youtube](#)

To test step 1 of the [homebrew cpu project](/cpu) let's see
if we can play an actual video from the EEPROM to the LCD character
display module. Bad Apple is a great video for this purpose: it's
essentially monochrome and most of the video is recognizable even
at a very low resolution.

The original Bad Apple video has 6569 frames at 30fps for a running
time of 3m39s. We have 32768 bytes in our EEPROM, so we need to run
our clock at 150 Hz to play the complete video in 3m39s.

This gives us about *5 bytes* of data per frame of video, or a
constant video bitrate of 1.2kbps.

![20 x 4 LCD display memory](/images/screen.svg)

The LCD character display module has 20 columns of characters and
4 rows. Each character is made of a 5x8 grid of pixels with a single
pixel space in between. The memory used to display characters is
split into two banks of 80 bytes interlaced so that lines 1 and 3
are bank 1 (D00-D39) and lines 2 and 4 are in bank 2 (E00-E39).

Bad Apple is in 4:3 aspect ratio which maps nicely to 40:32 pixels
given by taking the first 8 columns and 4 rows.

<img src="/images/character-codes.svg" height="400" alt="character ROM">

The character ROM on the LCD display module includes a "space"
character with none of the pixels set and a solid block character
(`'\xff'`) with all pixels set. These can be used to quickly paint
the solid parts of the video area.

![CGRAM memory](/images/cg-pattern.svg)

Custom character patterns are made possible with CGRAM. This display
module has 8 CGRAM characters we can use (CG0-CG7), so up to 1/4 of
the video area can display a unique pattern at a time.

CGRAM is contiguous but we label the addresses based on the
character and row, e.g. C30 is the first row of CG3. The pixel patterns
are given by the low 5 bits at each CGRAM address.

<img src="/images/commands.svg" height="400" alt="LCD commands">

We interface with the LCD display module using 8 data pins and a
register select (RS) pin. When RS is high we can send character codes
to display memory or data into CGRAM. When RS is low we can set the
display memory (DDRAM) or CGRAM address, clear the screen or issue
other cursor and display commands.

The EEPROM can only output 8 bits at a time but we need 9 bits including
RS to control the LCD display module. Let's rearrange the
table into a grid like the character codes above:

<img src="/images/command-grid.svg" height="400" alt="LCD command grid">

We can see there are some unused commands and commands with duplicate
values. If we add a lookup table that can store a value for RS and rewrite
the top 4 bits (i.e. move commands left or right in the table) then both
grids can be combined and represented with only 8 bits:

<img src="/images/hexmap.svg" height="400" alt="LCD command mapping">

Now we can issue one command per clock cycle from our EEPROM to our
LCD display module including:

- address all CGRAM
- address most of display memory (all of the video area)
- output ASCII(ish) characters and the solid block character `'\xff'`
- initialize the display with function set (`INI`)
- set the entry mode to "increment, no shifting" (`EIN`)
- clear the screen (`CLR`)
- hide the cursor (`HID`)

These are all we need to play our video and show text on the right
side of the display.

Thank you for reading this far. In the next post we'll explore the
software that encodes video onto the 32K EEPROM using these commands.
