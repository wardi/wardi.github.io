---
layout: post
title: "CMOS vs TTL: What are logic families?"
categories: [Electronics]
excerpt_separator: <!--more-->
---

<a href="https://youtu.be/1TpLYx1iZD4" class="yt-screen">
<img src="/images/logic-families.jpg" alt="Replaced 4011 with 7400">
</a>

In this video we replace our clock's 4011 CMOS Quad-NAND gate IC with a TTL equivalent and discuss compatibility between different logic families.

<!--more-->

## Transcript:

One nice thing about learning in public is that sometimes helpful strangers point out your mistakes for you.

The mistake I made a few videos back was to mix a CMOS chip, a quad nand gate, with TTL chips like a 74LS163.

Both families are 5 volts but the output level of a TTL high signal can be as low as 2.4 volts while CMOS chips expect at least 3.5 volts.

We can use CMOS with TTL as long as the CMOS is driving the TTL chip but in the clock module the halt signal is an input so this configuration doesn't work reliably.

So let's swap this CMOS quad nand for a TTL one.

Two of the gates are flipped around on the new chip so some of these connections need to be fixed.

Let's test the new chip by connecting the halt signal to the address pins to pause the animation.

Looks good to me.

Another benefit is the TTL chip pulls its inputs high so I can move the halt line and stay in the halted state until it's reconnected.

Short video this time but i hope you found it useful. Thanks for watching. See you next time.
