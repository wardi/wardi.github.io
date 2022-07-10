---
layout: page
title: 20xt6 Homebrew CPU Project
permalink: /cpu/
---

## Homebrew CPU Plan Step 1

<a href="https://youtu.be/QNkcTAgxSCc" class="yt-screen">
<img src="/images/step1.jpg" alt="Homebrew CPU Plan Step 1">
</a>

This video introduces the first milestone for our homebrew CPU project.

[transcript](/cpu-step-1/)

## Simple Clock Circuit with a 555 Timer

<a href="https://youtu.be/QfnkuXDf6NE" class="yt-screen">
<img src="/images/555.jpg" alt="Simple Clock Circuit with a 555 Timer">
</a>

In this video we create a clock circuit with a 555 timer in astable mode, and use it to blink a green LED.

[transcript](/clock-circuit/)

## Manual Clock Circuit Controls

<a href="https://youtu.be/LNIVcQHGDm4" class="yt-screen">
<img src="/images/clock_controls.jpg" alt="Manual Clock Circuit Controls">
</a>

We add a potentiometer and a single-step button to our clock circuit.

[schematic & transcript](/clock-controls/)

## Synchronous 4-bit Binary Counter 74LS163

<a href="https://youtu.be/U7ARbuAPPs4" class="yt-screen">
<img src="/images/74ls163.jpg" alt="Synchronous 4-bit Binary Counter 74LS163">
</a>

We connect our clock circuit to a 74LS163 synchronous binary counter and see how to count, reset and load values into the counter.

[transcript](/74ls163-counter/)

## 16-bit Address Register

<a href="https://youtu.be/FKlDwOu2p_8" class="yt-screen">
<img src="/images/16bit.jpg" alt="16-bit Address Register">
</a>

We add three more 74LS163's to our counter to make a 16-bit address register
for our homebrew CPU project.

[transcript](/16bit/)

## Power-on and Manual Reset Circuit

<a href="https://youtu.be/gnpy3CmJbko" class="yt-screen">
<img src="/images/reset.jpg" alt="Power-on and Manual Reset Circuit">
</a>

We add a reset circuit to our clock module that resets our address register to "0" on power-on and when a button is pressed.

[transcript](/reset/)

## Clock Halt Circuit with 4011 NAND Gate and 555 Timer

<a href="https://youtu.be/-j5fzLaksTk" class="yt-screen">
<img src="/images/halt.jpg" alt="Clock Halt Circuit with 4011 NAND Gate and 555 Timer">
</a>

We complete our clock module by adding a halt signal and test it with our address register. Now our (future) programs won't have to run forever.

[schematic & transcript](/halt/)

## SST39SF040 512k Flash Upgrade

<a href="https://youtu.be/UDAXxEo3heA" class="yt-screen">
<img src="/images/512k.jpg" alt="SST39SF040 512k Flash Upgrade">
</a>

We build a larger address register and use it to test an SST39SF040 512k flash chip.

[transcript](/512k-flash/)

## Milestone 1: LCD Character Display

<a href="https://youtu.be/neW9uogt1gw" class="yt-screen">
<img src="/images/milestone1.jpg" alt="LCD Character Display">
</a>

We connect an HD44780-powered LCD character display to the circuit and play a simple text animation from the program ROM.

[transcript](/milestone-1/)

## CMOS vs TTL: What are logic families?

<a href="https://youtu.be/1TpLYx1iZD4" class="yt-screen">
<img src="/images/logic-families.jpg" alt="Replaced 4011 with 7400">
</a>

In this video we replace our clock's 4011 CMOS Quad-NAND gate IC with a TTL equivalent and discuss compatibility between different logic families.

[transcript](/logic-families/)
