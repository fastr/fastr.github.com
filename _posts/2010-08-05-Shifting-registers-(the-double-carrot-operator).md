---
layout: article
title: Shifting registers (the double carrot operator)
categories: c embedded
updated_at: 2010-08-05
---

Goal
====

Explain the use of `<<` and `|` in `C` and the linux kernel. Intended for those with basic knowledge of computer science and computer engineering.

Status
------

Someone with more experience should look this over. There are goof-O's and misexplanations still.

TODO use an actual example examining just the first 8 bits

Multiplexing
=============

Let's consider TI's OMAP3530 Application Processor.
There are a lot of GPIO (general-purpose i/o) pins (which can be used as just about anything by bit-banging).
Many of them are multiplexed meaning that it can be used as a gpio pin or part of the camera data line interface, but not both at the same time.

For example:
The gumstix has an optionl LCD screen. Their driver for that screen uses lines that could otherwise be used as SPI (serial peripheral interface - a poor man's USB).

Instead of giving enough lines to do every possible thing under the sun at all times, there are a limited number of lines drawn out from the processor.
If you want to use one feature, you may not be able to use another.

There has to be a setting that tells the processor which line is being used for what.

What's a Register?
==================

A register holds a series of bits that the processor (or device) uses to store information or configuration.

The ISP / CCDC interface for the OMAP35x has numerous settings.
It can be configured to capture raw data, jpeg data, or other specially formatted data from a camera device.
It can be configured to interpret the vsync and hsync signals needed for a motion camera to separate data into frames
or to produce those signals.

Many of these settings are simply on or off. A few of them are enumerable (raw or rgb or jpeg or yuv mode).

If each setting took one byte of storage, that would be a lot of wasted bits.

Many registers are 32-bits and so they can store lots of this settings information.

Let's consider a simple 8-bit register:

    00000000

If this were the settings register for a device with 2 binary settings and 2 enumerable settings, one with 5 options, another with 3, then the default settings would be `0x0` by default.

Let's say the bits have this mapping:

    * [0] VSYNO vsync out - 0 on, 1 off
    * [1] HSYNO hsync out - 0 on, 1 off
    * [2-4] capture mode - (3 bits mean a max of 8 options, we'll use just 5)
      * 000 RAW
      * 001 JPEG
      * 010 YUV
      * 011 RGB
      * ... so on
    * [5-6] mask a color (r, g, or b)
      * 00 MNONE
      * 01 MRED
      * 10 MGREEN
      * 11 MBLUE
    * [7] reserved for future use / wasted

This is how we might represent that vsync and hsync are outputs, the mode is raw, and we want to mask the color green (I don't know why, but just for example's sake):

    01000011

No problem right? Any time you want to set that setting you could quickly come up with the decimal representation inside of your head and assign that to the address location, right?

Setting Registers in C
======================

To simplify setting the register above you might create a header like this:

camera_reg.h

    // a long list of registers (just 2 for our example)
    #define CAMERA_REG_BASE 0x0034 C600 // Where all of the CAMERA device registers might begin
    #define CAMERA_CAP_CFG_OFFSET 0x00  // How far away from the BASE the CAP_CFG register is. This would be the very first
    #define CAMERA_OUT_CFG_OFFSET 0x08  // The next register, 8 bits away
    
    // Which bit of the 8 sets which setting
    #define CCAPCFG_HSYNC_SHIFT 1        // 0000 0001
    #define CCAPCFG_VSYNC_SHIFT (1 << 1) // 0000 0010
    #define CCAPCFG_MODE_SHIFT  (1 << 2) // 0000 0100
    #define CCAPCFG_MASK_SHIFT  (1 << 5) // 0001 0000
    #define CCAPCFG_RSV_SHIFT   (1 << 7) // 0100 0000
    
    
    #define CCAPCFG_HSYNC_EN 1
    #define CCAPCFG_VSYNC_EN 2
    #define CCAPCFG_MODE_JPEG 4
    #define CCAPCFG_MODE_YUV 8
    #define CCAPCFG_MODE_RGB 12
    // ...
    #define CCAPCFG_MASK_RED 32
    #define CCAPCFG_MASK_GREEN 64
    #define CCAPCFG_MASK_BLUE 96

And then address the register in this fashion:

    *(void*)(CAMERA_REG_BASE | CAMERA_CAP_CFG_OFFSET) = (CCAPCFG_HSYNC_EN|CCAPCFG_VSYNC_EN|CAPCFG_MASK_GREEN);
