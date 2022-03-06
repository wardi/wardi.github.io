---
layout: page
title: Video Encoding at 5 bytes/frame
permalink: /video-encoding/
---

[![Bad Apple on 32K EEPROM](/images/arms-wide.jpg)
Watch "Bad Apple on 32K EEPROM" on Youtube](https://youtu.be/TkhVqe8Z2lY)

This is part two of a two-part series. [Read part one here](/bad-apple).

In this post we encode the full Bad Apple video into 32 kilobytes using
an 8-bit lookup table driving an HD44780 display.
We also juggle 8 CGRAM characters across an 8 x 4 video area
to simulate a full bitmap display.

<!--more-->

<style>.highlight {line-height:1.2}</style> <!--improve braille-pixel appearance-->

## Encoder Files

<img src="/images/encoder-files.svg" width="450" alt="Encoder Files">

The encoder is made of a number of Python scripts, some that generate
other Python scripts.

- [`extract.py`](https://github.com/wardi/cpu/blob/main/bad-apple/extract.py)
  uses OpenCV to convert the `badapple.mp4` input video to a bitmap format suitable for
  encoding `badapple.enc.gz`.
- [`lookup_table.py`](https://github.com/wardi/cpu/blob/main/bad-apple/lookup_table.py)
  generates the lookup table binary `ltable.bin` and a Python module with mnemonics
  for values in the table
  [`baconsts.py`](https://github.com/wardi/cpu/blob/main/bad-apple/baconsts.py).
- [`encoder.py`](https://github.com/wardi/cpu/blob/main/bad-apple/encoder.py)
  encodes `badapple.enc.gz` into a Python script that exports the video binary file
  `video.bin` using the table mnemonics from `baconsts.py`.

## Lookup Table

[`lookup_table.py`](https://github.com/wardi/cpu/blob/main/bad-apple/lookup_table.py)
contains a text version of the command lookup table:

```python
# byte order from top->bottom, left->right
# (mnemonic:)hex-value
hex_map = """
C00:04 C20:05 12 13 14 15 16 17 D00:08 D16:09 D32:0a C40:06 E00:0c E16:0d E32:0e C60:07
CLR:00 C21:05 12 13 14 15 16 17 D01:08 D17:09 C01:04 C41:06 E01:0c E17:0d E33:0e C61:07
HOM:00 C22:05 12 13 14 15 16 17 D02:08 D18:09 C02:04 C42:06 E02:0c E18:0d E34:0e C62:07
  R:01 C23:05 12 13 14 15 16 17 D03:08 D19:09 C03:04 C43:06 E03:0c E19:0d E35:0e C63:07
...
```

The optional mnemonic labels a value and the hex value is the
value stored at that location in the table. The low 4 bits of the hex value are
connected to the high data lines of the HD44780 controller and the next bit is connected
to the RS line.

Let's look at two examples and compare with the original command lookup table:

<img src="/images/hexmap-top.svg" width="510" alt="LCD command mapping">

The second position on the left "`CLR:00`" is offset `0x01` in order from top to bottom:
- has a mnemonic of `CLR`
- has a hex value of `0x00` which means "set RS to 0 and D4-D7 to `0b0000`"
- D0-D3 are copied from the low bits of the offset value `0b0001`
- sending this command will clear the screen and reset the cursor position
  on the LCD display

The third position in the fifth column "`14`" is offset `0x42` in order from top to bottom:
- has no mnemonic but corresponds to ASCII character "`B`"
- has a hex value of `0x15` which means "set RS to 1 and D4-D7 to `0b0100`"
- D0-D3 are copied from the low bits of the offset value `0b0010`
- sending this command will output a "`B`" character and advance the cursor
  on the LCD display

All mnemonic offset values are exported from this table as simple variables in a
Python module
[`baconstants.py`](https://github.com/wardi/cpu/blob/main/bad-apple/baconstants.py):

```python
C00 = b"\x00"
CLR = b"\x01"
HOM = b"\x02"
CRT = b"\x03"
...
```

## Initialization

The video is encoded into a Python script `video.py` that will generate the video
binary file `video.bin`. On startup the LCD display needs to be initialized, so the
script starts with:

```python
from baconstants import *

with open('video.bin', 'wb') as f:
    f.write(
        INI + # function set: initial setup
        HID + # hidden cursor
        EI0 + # entry incrementing, no shift
        CLR
    )
...
```

The `INI` command sets the LCD display into 8-bit mode. This can only be done
once after initial powerup and is ignored afterwards. `HID` enables the display
but disables the blinking and underline cursor modes. `EI0` disables screen
shifting on output and makes the cursor increment to the next location after
each character is written.

## Frame Comparison

While encoding the video we output a frame comparison and some bookkeeping
information on every new frame as comments in `video.py`. These comments
let us more easily find bugs or places for improvement without needing
to play the video on real hardware:

```python
...
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣷⣄⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣷⡀⠀ frame 3278
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠙⣿⡇⢻⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠙⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡆ bytes sent 16557
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⡖⠀⡶⠖⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⡖⠀⡴⠖⠂⣶⣶⡆⣶⣶⡆ position 32
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⡏⠁⠀⠀⠂⠠⠾⠃⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⡉⠁⠀⠀⠂⠠⠶⠃⣿⣿⡇⣿⣿⡇ delta 24
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⠀⠀⠀⠀⣤⣬⡅⣭⣭⡅⣭⣭⡅ » ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣥⠀⠀⠀⠀⣤⣬⡅⣭⣭⡅⣭⣭⡅ ▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⠛⠋⠀⠀⠀⡀⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⠛⠋⠀⠀⢀⡀⣿⣿⡇⣿⣿⡇⣿⣿⡇ cgram 8/8
# ⣛⣛⡃⣛⣛⡃⣛⠛⠁⠀⠀⠀⠀⠚⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⣛⣛⡃⣛⣛⡃⠛⠛⠃⠀⠀⠀⠘⠛⠃⣛⣛⡃⣛⣛⡃⣛⣛⡃ {24: 5, 15: 1, 13: 6, 23: 2, 14: 0, 7: 3, 4: 7, 31: 4}
# ⣿⣿⡇⣿⣿⡇⠃⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⡇⣿⣿⡇⣿⣿⡇ .
# ⠿⠿⠇⠿⠟⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠽⠿⠇⠿⠿⠇⠿⠿⠇ » ⠿⠿⠇⠿⠏⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠿⠿⠇⠿⠿⠇⠿⠿⠇ .
...
```

On the left is the ideal frame from `badapple.enc.gz`, on the right is what
would be displayed on the LCD display at this point.
`frame` is the frame number, `bytes sent` is the position within the file,
`position` is the cursor position on the display, `delta` is the number
of pixels different between ideal and display frames,
`cgram` is the number of CGRAM characters in use and the mapping
of screen positions to CGRAM character indexes.

These comments are responsible for `video.py` being more than 10MB
so it's not stored in the same repo. 👉 Run `encoder.py badapple.enc.gz`
to generate `video.py` on your own machine.

## Drawing Boxes

Drawing a blank space or solid block character to any location in the
video area is one of the simplest updates. If the cursor is not already
in position we first send a cursor movement command e.g. `E00` to move
to the first character in the second row.

Next send a space character `b' '` to turn off all pixels in a 5x8 cell or a
solid block character `b'\xff'` to turn them all on.

At the start of the video we need to draw an all-black screen (all pixels on)
so the first few frames are spent drawing boxes:
```python
...
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀ frame 0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀ bytes sent 6
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ position 6
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 1040
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
    f.write(b'\xff')
    f.write(b'\xff')
    f.write(E00)
    f.write(b'\xff')
    f.write(b'\xff')
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 1
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 11
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ position 12
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 880
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⠉⠉⠁⠉⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
    f.write(b'\xff')
    f.write(b'\xff')
    f.write(b'\xff')
    f.write(b'\xff')
    f.write(b'\xff')
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 2
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 16
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⠀⠀ position 17
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀ delta 680
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
...
```
