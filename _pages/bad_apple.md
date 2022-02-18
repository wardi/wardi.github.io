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

The original Bad Apple video has 6569 frames at 30 fps for a running
time of 3m39s. We have 32768 bytes so we need to run
our clock at 150 Hz to fit the complete video in EEPROM.

This gives us about *5 bytes* of data per frame of video, or a
constant video bitrate of 1.2 kb/s.

## LCD display memory

The LCD character display module uses an HD44780 controller and
has 20 columns of characters and 4 rows of character cells.
Each character cell is a 5 x 8 grid of pixels with a single
pixel space in between.

<img src="/images/screen.svg" height="200" alt="20 x 4 LCD display memory">

The memory used to display characters is
split into two banks of 80 bytes interlaced so that lines 1 and 3
are bank 1 (D00-D39) and lines 2 and 4 are in bank 2 (E00-E39).

Bad Apple is in 4:3 aspect ratio which maps nicely to 40:32 pixels
given by taking the first 8 columns and 4 rows.

<img src="/images/character-codes.svg" height="500" alt="character ROM">

The character ROM on the LCD display module includes a "space"
character with none of the pixels set and a solid block character
(`'\xff'`) with all pixels set. These can be used to quickly paint
the solid parts of the video area.

## CGRAM

Custom character patterns are made possible with CGRAM. This display
module has 8 CGRAM characters we can use (CG0-CG7), so up to 1/4 of
our 8 x 4 video area character cells can display a unique pattern at once.

![CGRAM memory](/images/cg-pattern.svg)

The pixel patterns are given by the low 5 bits stored at each CGRAM address.
CGRAM is contiguous but we number the addresses based on the
character and row. E.g. C10 is the first row of CG1, and follows immediately
after C07.

## LCD commands

We interface with the LCD display module using 8 data pins (DB0-DB7)
and a register select (RS) pin.

<img src="/images/commands.svg" height="500" alt="LCD commands">

When RS is high we can send character codes
to display memory or data into CGRAM. When RS is low we can set the
display memory (DDRAM) or CGRAM address, clear the screen or issue
other cursor and display commands.

Rearranging the table into a grid like the character codes above
we can see there are some unused commands and commands with duplicate
values:

<img src="/images/command-grid.svg" height="500" alt="LCD command grid">

## Command compression

The EEPROM can only output 8 bits at a time but we need 9 bits including
RS to fully control the LCD display module.

If we add a 256-byte lookup table for RS then
we can combine sending character codes and LCD commands into 8 bits.
If we also reassign the upper 4 bits (DB4-DB7) we can assign new positions
for commands in the table.

Reassigning the upper 4 bits is the same as moving a command left or
right in the table.

![Lookup table circuit](/images/behind-lcd.jpg)

This is the combined "decompression" table that maps the 8 input bits from
the EEPROM to 9 output bits for the LCD display module:

<img src="/images/hexmap.svg" height="400" alt="LCD command mapping">

This table allows issuing one command per clock cycle from our EEPROM to our
LCD display module including:

- addressing all CGRAM
- addressing most of display memory (all of the video area)
- output ASCII(ish) characters and the solid block character `'\xff'`
- output CGRAM characters (CG0-CG7)
- initialize the display with function set (`INI`)
- set the entry mode to "increment, no shifting" (`EIN`)
- clear the screen (`CLR`)
- hide the cursor (`HID`)

These are all we need to play our video and show text on the right
side of the display.

Thank you for reading this far. In the next post we'll explore the
software that encodes video onto the 32K EEPROM using these commands.
