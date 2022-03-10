---
layout: page
title: Video Encoding at 5 bytes/frame
permalink: /video-encoding/
---

[![Bad Apple on 32K EEPROM](/images/arms-wide.jpg)
Watch "Bad Apple on 32K EEPROM" on Youtube](https://youtu.be/TkhVqe8Z2lY)

This is part two of a two-part series. [Read part one here](/bad-apple).

In this post we encode the full Bad Apple video into 32 kilobytes using
an 8-bit lookup table to drive an HD44780-powered LCD display. We have
no CPU so our video "compression" is limited to the features of the
LCD display.

To make it look like we have more than 8 CGRAM characters
we carefully juggle them across the 8 x 4 video area and lean on LCD
display persistence for the illusion of a full bitmap display.

<!--more-->

<style>.braille-pixels + .language-python {line-height:1.108}</style>

## Encoder Files

<img src="/images/encoder-files.svg" width="450" alt="Encoder Files">

The encoder is made of a number of Python scripts, some that generate
other Python scripts.

- [extract.py](https://github.com/wardi/cpu/blob/main/bad-apple/extract.py)
  uses OpenCV to convert the `badapple.mp4` input video to a bitmap format suitable for
  encoding `badapple.enc.gz`.
- [lookup_table.py](https://github.com/wardi/cpu/blob/main/bad-apple/lookup_table.py)
  generates the lookup table binary `ltable.bin` and a Python module with mnemonics
  for values in the table
  [baconsts.py](https://github.com/wardi/cpu/blob/main/bad-apple/baconsts.py).
- [encoder.py](https://github.com/wardi/cpu/blob/main/bad-apple/encoder.py)
  encodes `badapple.enc.gz` into a Python script that exports the video binary file
  `video.bin` using the table mnemonics from `baconsts.py`.

## Lookup Table

[lookup_table.py](https://github.com/wardi/cpu/blob/main/bad-apple/lookup_table.py)
contains a text version of the [command lookup table](/bad-apple/#command-lookup-table):

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># byte order from top-&gt;bottom, left-&gt;right
# (mnemonic:)hex-value
</span><span class="n">hex_map</span> <span class="o">=</span> <span class="s">"""
C00:04 C20:05 12 13 14 15 16 17 D00:08 D16:09 D32:0a C40:06 E00:0c E16:0d E32:0e C60:07
<span class="c1">CLR:00</span> C21:05 12 13 14 15 16 17 D01:08 D17:09 C01:04 C41:06 E01:0c E17:0d E33:0e C61:07
HOM:00 C22:05 12 13 <span class="c1">14</span> 15 16 17 D02:08 D18:09 C02:04 C42:06 E02:0c E18:0d E34:0e C62:07
  R:01 C23:05 12 13 14 15 16 17 D03:08 D19:09 C03:04 C43:06 E03:0c E19:0d E35:0e C63:07
...
</span></code></pre></div></div>

The optional mnemonic labels a value and the hex value is the
value stored at that location in the table. The low 4 bits of the hex value are
connected to the high data lines of the HD44780 controller and the next bit is connected
to the RS line.

Let's look at two examples and compare with the original
[command lookup table](/bad-apple/#command-lookup-table):

<img src="/images/hexmap-top.svg" width="510" alt="LCD command mapping">

The second position on the left "`CLR:00`" is offset `0x01` in order from top to bottom:
- has a mnemonic of `CLR`
- has a hex value of `0x00` which means "set RS to 0 and D4-D7 to `0b0000`"
- D0-D3 are copied from the low bits of the offset value `0b0001`
- sending this command will clear the screen and reset the cursor position
  on the LCD display

The third position in the fifth column "`14`" is offset `0x42` in order from top to bottom:
- has no mnemonic but corresponds to ASCII character "`B`"
- has a hex value of `0x14` which means "set RS to 1 and D4-D7 to `0b0100`"
- D0-D3 are copied from the low bits of the offset value `0b0010`
- sending this command will output a "`B`" character and advance the cursor
  on the LCD display

All mnemonic offset values are exported from this table as simple variables in a
Python module
[baconsts.py](https://github.com/wardi/cpu/blob/main/bad-apple/baconsts.py):

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
from baconsts import *

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

<div class="braille-pixels"></div>

```python
...
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣷⣄⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣷⡀⠀ frame 3278
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠙⣿⡇⢻⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠙⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡆ bytes sent 16557
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⡖⠀⡶⠖⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⡖⠀⡴⠖⠂⣶⣶⡆⣶⣶⡆ position E22
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
so it's not stored in the same repo.

👉 Follow along by generating `video.py` on your own machine:

```bash
$ python encoder.py badapple.enc.gz > video.py
```

## Drawing Blocks

Drawing a blank space or solid block character to any location in the
video area is one of the simplest updates. If the cursor is not already
in position we first send a [cursor movement command](/bad-apple/#lcd-display-memory)
e.g. `E00` to move to the first character in the second row.

Next send a space character `b' '` to turn off all pixels in a 5x8 cell or a
solid block character `b'\xff'` to turn them all on.

At the start of the video we need to draw an all-black screen (all pixels on)
so the first few frames are spent drawing blocks:

<div class="braille-pixels"></div>
<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">...</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » <span class="s">⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇</span>⠀⠀⠀⠀⠀⠀ frame 0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » <span class="s">⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇</span>⠀⠀⠀⠀⠀⠀ bytes sent 6
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ position D06
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 1040
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">E00</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇<span class="s">⣿⣿⡇⣿⣿⡇</span> frame 1
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇<span class="s">⣿⣿⡇⣿⣿⡇</span> bytes sent 11
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » <span class="s">⣶⣶⡆⣶⣶⡆</span>⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ position E02
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » <span class="s">⣿⣿⡇⣿⣿⡇</span>⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 880
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » <span class="s">⠉⠉⠁⠉⠉⠁</span>⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\xff</span><span class="s">'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 2
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 16
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆<span class="s">⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆</span>⠀⠀⠀ position E07
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇<span class="s">⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇</span>⠀⠀⠀ delta 680
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⠉⠉⠁⠉⠉⠁<span class="s">⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁⠉⠉⠁</span>⠀⠀⠀ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ {}
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
</span><span class="p">...</span>
</code></pre></div></div>

## Clearing the screen

When enough of the screen pixels need to be switched from on to off
we can clear the whole screen with a single `CLR` command:

<div class="braille-pixels"></div>

```python
...
# ⢿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 789
# ⠀⠉⠃⠿⣿⡇⣿⣿⡇⣿⡟⠁⠙⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⡟⠁⠛⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 3989
# ⠀⠀⠀⠀⠀⠀⠲⢶⡆⡒⠀⡀⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⠂⠀⡀⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ position E23
# ⠀⠀⠀⠀⠀⠀⠀⠀⠁⠓⠶⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣶⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ delta 559
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠩⠅⣭⣭⡅⣭⣭⡅⣭⣭⡅ » ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅ ▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴▴
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠛⠇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ cgram 3/8
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠙⠃⢛⣛⡃ » ⠛⠛⠃⠛⠛⠃⠛⠛⠃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ E03:CG4 D04:CG3 D03:CG0
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠃ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ .
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ .
    f.write(CLR)
    f.write(D05)
    f.write(b'\xff')
    f.write(b'\xff')
    f.write(b'\xff')
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠛⠇⢿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 790
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⡇⡟⠙⠃⢿⣿⡇⣿⣿⡇⣿⣿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 3994
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠠⣶⡆⠀⠀⠀⠀⠀⠂⠶⣶⡆⣶⣶⡆ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ position D08
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠁⠻⢿⡇ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 103
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠁ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ ▴▴▴▴▴▴
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ cgram 0/8
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
# ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ .
...
```

This can prevent falling too far behind on screen updates when large
parts of the screen are changing, but is distracting if overused.

We strike a balance by only clearing the screen when at least
[half of the pixels displayed are wrong](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L298)
and the number of wrong pixels is at least 1.5 times the number of
pixels that should be "on".

## Drawing Pixels

Drawing pixels on the screen requires:
- moving the cursor to the [CGRAM area](/bad-apple/#cgram)
  e.g. `C20` for the first line of CGRAM character `CG2`
- filling 8 bytes of data, only the least significant 5 bits
  matter so we use commands `b'@'` (all 0s) through `b'_'` (all 1s)
  in the ASCII command range
- moving the cursor to the [desired location](/bad-apple/#lcd-display-memory)
  e.g. `E27` for the bottom-right corner of the screen
- writing the CGRAM character to display the CGRAM pattern at
  this location e.g. `CG2`


<div class="braille-pixels"></div>
<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">...</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">C20</span><span class="p">)</span> <span class="c1"># assign CG2 to E27 (9 steps)
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'@'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠃⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠃⠀⠀⠀⠀⠀⠀⠀⠀ frame 23
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ bytes sent 122
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀ position C21
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 22
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣤⣤⡄⠀⠀⠀⠀⠀⠀ ▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣷⡆⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀ cgram 2/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣃⠀⠀⠀⠀⠀ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣀⠀⠀⠀⠀⠀ E26:CG1 D05:CG0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣆⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠆⠀⠀⠀ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠆⠀⠀⠀ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'@'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'@'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'@'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'@'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'X'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⡟⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠃⠀⠀⠀⠀⠀⠀⠀⠀ frame 24
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⠇⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ bytes sent 127
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀ position C26
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠟⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 35
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣥⣤⡀⠀⠀⠀⠀⠀⠀ » ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣤⣤⡄⠀⠀⠀⠀⠀⠀ ▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣦⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀ cgram 2/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⡃⠀⠀⠀⠀ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣀⠀⠀⠀⠀⠀ E26:CG1 D05:CG0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣇⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣆⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠦⠀⠀ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠆⠀⠀⠀ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'X'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'</span><span class="se">\\</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">E27</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">CG2</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">E26</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⠃⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠃⠀⠀⠀⠀⠀⠀⠀⠀ frame 25
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠁⠀⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ bytes sent 132
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣖⡀⣀⡀⠀⠀⠀⠀⠀⠀⠀ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀ position E26
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⡃⠀⠀⠀⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀ delta 63
# ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡄⣄⠀⠀⠀⠀⠀ » ⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣭⣭⡅⣤⣤⡄⠀⠀⠀⠀⠀⠀ ▴▴▴
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣧⠀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⠀⠀⠀⠀⠀⠀ cgram 3/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⠀⠀⠀⠀ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣀⠀⠀⠀⠀⠀ E26:CG1 D05:CG0 E27:CG2
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡀⠀⠀⠀ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣆⠀⠀⠀⠀ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠄⠀ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠆<span class="s">⠿⠄</span>⠀ .
</span><span class="p">...</span>
</code></pre></div></div>


Notice that we've spent about two frames (11 bytes) updating
just a single character cell. That's very slow if need to update
the whole screen this way.

## Optimization strategies

We mitigate the slow speed of pixel updates a few different ways:
- do all screen clearing and
  [block updates first](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L320)
  (just 1 or 2 bytes per character cell)
- update based on the state [a couple frames ahead](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L345)
- prefer to not update cells that are only a
  [few pixels different](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L349)
  than "all-on" or "all-off" (block updates are fine for these)
- don't update cells that will become all-on or all-off
  [in a few frames](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L351)
- choose the unassigned CGRAM character that has the
  [smallest difference](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L405)
  to save a few bytes updating pixel data
- when updating a cell already assigned to a CGRAM character,
  [update the pixel data in-place](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L381)

Example in-place pixel update of CGRAM character `CG0` at
position `E04` (the witch's back and hat) in only 7 bytes:

<div class="braille-pixels"></div>
<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">...</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 571
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 2889
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀⠐⢶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀<span class="s">⠐⢶⡆</span>⣶⣶⡆⣶⣶⡆⣶⣶⡆ position C66
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⠀⠐⠸⠇⠿⠿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣷⡁<span class="s">⠀⠘⠇</span>⠿⠿⡇⣿⣿⡇⣿⣿⡇ delta 8
# ⣭⣭⡅⣭⣭⡅⣭⣤⡄⣤⣤⡄⣄⣠⡄⣀⣀⡀⣭⣭⡅⣭⣭⡅ » ⣭⣭⡅⣭⣭⡅⣭⣤⡄⣤⣤⡄⣄⣠⡄⣤⣀⡄⣭⣭⡅⣭⣭⡅
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ cgram 6/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ E05:CG2 D25:CG4 E04:CG0 E02:CG3 D24:CG7 E03:CG6
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'^'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'D'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">C00</span><span class="p">)</span> <span class="c1"># update assigned CG0 at E04
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'G'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'O'</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 572
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 2894
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀⠐⢶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀<span class="s">⠠⢶⡆</span>⣶⣶⡆⣶⣶⡆⣶⣶⡆ position C02
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⠀⠐⠸⠇⡿⢿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣷⠁<span class="s">⠀⠘⠇</span>⠿⠿⡇⣿⣿⡇⣿⣿⡇ delta 12
# ⣭⣭⡅⣭⣭⡅⣭⣤⡄⣤⣤⡄⣄⣠⡄⣀⣀⡀⣭⣭⡅⣭⣭⡅ » ⣭⣭⡅⣭⣭⡅⣭⣤⡄⣤⣥⡄⣄⣠⡄⣤⣀⡄⣭⣭⡅⣭⣭⡅
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ cgram 6/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ E05:CG2 D25:CG4 E02:CG3 D24:CG7 E03:CG6 E04:CG0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ .
</span>    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'G'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">C04</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'K'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="sa">b</span><span class="s">'C'</span><span class="p">)</span>
    <span class="n">f</span><span class="p">.</span><span class="n">write</span><span class="p">(</span><span class="n">E05</span><span class="p">)</span>
<span class="c1"># ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ frame 573
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ bytes sent 2899
# ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀⠰⢶⡆⣶⣶⡆⣶⣶⡆⣶⣶⡆ » ⣶⣶⡆⣶⣶⡆⣶⣶⡆⣶⣄⠀<span class="s">⠠⣶⡆</span>⣶⣶⡆⣶⣶⡆⣶⣶⡆ position E05
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣷⠁⠐⠸⠇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣷⠁<span class="s">⠐⠸⠇</span>⠿⠿⡇⣿⣿⡇⣿⣿⡇ delta 11
# ⣭⣭⡅⣭⣭⡅⣥⣤⡄⣤⣥⡄⣄⣠⡄⣀⣀⡁⣭⣭⡅⣭⣭⡅ » ⣭⣭⡅⣭⣭⡅⣭⣤⡄⣤⣥⡄⣄⣠⡄⣤⣀⡄⣭⣭⡅⣭⣭⡅
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ cgram 6/8
# ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ » ⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃⣛⣛⡃ E05:CG2 D25:CG4 E02:CG3 D24:CG7 E03:CG6 E04:CG0
# ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ » ⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇⣿⣿⡇ .
# ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ » ⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇⠿⠿⠇ .
</span><span class="p">...</span>
</code></pre></div></div>

# Up in the air

When pixel updates in the video are constrained to 8 character cells the
illusion of full screen video is very convincing:

![8 active character cells](/images/corners.jpg)

But when more than 8 character cells needing pixel updates we start juggling
CGRAM characters by
[evicting the oldest](https://github.com/wardi/cpu/blob/963e6843e24dcffaa64e349e125b04c109824200/bad-apple/encoder.py#L421)
character cell and moving its CGRAM character to a new location on screen.

The LCD character display has "display persistence" meaning it hangs on
to what was previously displayed for a short time. This effect gives us some
cover when juggling CGRAM characters:

![More than 8 active character cells](/images/umbrella.jpg)


