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

## Encoder Files

<img src="/images/encoder-files.svg" height="300" alt="Encoder files">
![Encoder files](/images/encoder-files.svg)

The encoder is made of a number of Python scripts, some that generate
other Python scripts.

- [`extract.py`](https://github.com/wardi/cpu/blob/main/bad-apple/extract.py)
  uses OpenCV to convert the mp4 input video to a bitmap format suitable for encoding.
- [`lookup_table.py`](https://github.com/wardi/cpu/blob/main/bad-apple/lookup_table.py)
  generates the lookup table binary `ltable.bin` and a python module with mnemonics
  for values in the table
  [`baconstants.py`](https://github.com/wardi/cpu/blob/main/bad-apple/baconstants.py)
- [`encoder.py`](https://github.com/wardi/cpu/blob/main/bad-apple/encoder.py)
  encodes the video into a python script that exports the video binary file
  `video.bin` using the table mnemonics.

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
CRT:01 C23:05 12 13 14 15 16 17 D03:08 D19:09 C03:04 C43:06 E03:0c E19:0d E35:0e C63:07
...
```

Where the (optional) mnemonic labels a value in the table and the hex value is the
value stored at that location in the table. The low 4 bits of the hex value are
connected to the high data lines of the HD44780 controller and the next bit is connected
to the RS line. Let's look at three examples:

The first position in the top left is position 0, which has:
- a mnemonic of `C00`
- a value of `0x04` which means "set RS to 0 and D4-D7 to `0b0100`"
- D0-D3 are copied from the low bits of the data value `0b0000`
- sending this data value of `0` will move to CGRAM character 0, line 0

The next position below the last is position 1, which has:
- a mnemonic of `CLR`
- a value of `0x00` which means "set RS to 0 and D4-D7 to `0b0000`"
- D0-D3 are copied from the low bits of the data value `0b0001`
- sending this byte value of `1` will clear the screen and reset the cursor

The first position in the 6th column is position 80, which has:
- no mnemonic but corresponds to ASCII character "P"
- a value of `0x15` which means "set RS to 1 and D4-D7 to `0b0101`"
- D0-D3 are copied from the low bits of the data value `0b0000`
- sending this byte value of `80` will output a "`P`" character and advance the cursor

All mnemonics are exported as simple variables in a python module
[`baconstants.py`](https://github.com/wardi/cpu/blob/main/bad-apple/baconstants.py):

```python
C00 = b"\x00"
CLR = b"\x01"
HOM = b"\x02"
CRT = b"\x03"
...
```

## Initialization

```python
with open('video.bin', 'wb') as f:
    f.write(
        INI + # function set: initial setup
        HID + # hidden cursor
        EI0 + # entry incrementing, no shift
        CLR
    )
...
```

## Drawing Boxes

A simple screen update we can perform with our commands is to draw
a blank space or solid block character to any location in the video
area. This updates a complete 5x8 pixel cell to all-off or all-on.
and involves sending
a command 

