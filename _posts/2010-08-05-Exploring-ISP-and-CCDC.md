---
layout: article
title: Exploring ISP and CCDC
categories: uncategorized
updated_at: 2010-08-05
---

Goal
====

Trace from a hardware CCDC register setting through the ISP / CCDC module, through the driver, through V4L2, right to the application interface.

This example traces from the hardware bit for interlace mode in the CCDC regisetrs.

Warning: this is just thought-jotting. Not well organized yet.

Register Names
===========

The `#define`s in the kernel are very nearly 1:1 mapped with the documentation in the TRM (spruf98g).

For example: `CCDC_SYN_MODE[7] FLDMODE` in the TRM is `ISPCCDC_SYN_MODE_FLDMODE` in the kernel. Almost exactly the same.

My general rule is to grep the most unique part of the name in question from the root of the kernel directory.

For example: `grep FLDMODE -R ./`


Interlaced Mode
===========

The device is running in interlace mode even with `CCDC_SYN_MODE[7] FLDMODE` set to 0. Tracing...

Where is interlace mode set?
----------------

`grep FLDMODE -R ./` shows

it is defined in `./drivers/media/video/isp/ispreg.h` 

  #define ISPCCDC_SYN_MODE_FLDMODE    (1 << 7)


and used in `./drivers/media/video/isp/ispccdc.c`

    if (syncif.fldmode)
      syn_mode |= ISPCCDC_SYN_MODE_FLDMODE;
    else
      syn_mode &= ~ISPCCDC_SYN_MODE_FLDMODE;


What is `syn_mode`?
-----------------

It appears to be the value as read from the 32-bit isp register

    u32 syn_mode = isp_reg_readl(isp_ccdc->dev, OMAP3_ISP_IOMEM_CCDC,
               ISPCCDC_SYN_MODE);

What values it is set to depend heavily upon `syncif` (of type `ispccdc_syncif`) and then it is written back to the register.


What is `syncif`?
----------------

A struct defined in `./drivers/media/video/isp/ispccdc.h`

It is set according to `pipe->ccdc_in`, which is an enum used to describe a format such as `CCDC_RAW_GBRG` (in the same file).

If the value is `CCDC_OTHERS`, `syncif` is not set, which means that the values already in the register stay in the register?

Not necessarily; since the ISP only knows about certain `CCDC_XXX_XXXX` formats it will unset certain registers for all `CCDC_OTHERS` formats.

However, it appears that `CCDC_OTHERS` and `DAT12` are reserved for future use. `CCDC_OTHERS` isn't handled in `isp_try_pipeline`.


Where does `pipe->ccdc_in` come from?
----------------

It is of type `isp_pipeline` and the `ccdc_in` comes from `./drivers/media/video/isp/isp.c`

It is determined by `pix_input->pixelformat`


What's the deal with `pix_input->pixelformat`?
--------------

It's a V4L format, such as `V4L2_PIX_FMT_UYVY`, as set in `./drivers/media/video/your_driver_here.c`

The formats supported by the OMAP ISP are defined in `./drivers/media/video/isp/isp.c`.


Where would I define it in `./drivers/media/video/your_driver_here.c`
--------------

hmm... various places... I don't understand this stack well enough to say yet...


Where would I find these `V4L2_PIX_FMT_XXX`?
--------------

`./include/linux/videodev2.h`

And, fancy this - Documentation: `./Documentation/DocBook/v4l/videodev2.h.xml`

How would I create my own `V4L2_PIX_FMT_XXX`?
--------------

In `./include/linux/videodev2.h` there's a special place for that

    /*  Vendor-specific formats   */
    #define V4L2_PIX_FMT_CUSTOM v4l2_fourcc('C','F','M','T') // Arbitrary 4-characters which must be unique among V4L defines

For quick reference: `V4L2_PIX_FMT_XXX` is a u32

    /*  Four-character-code (FOURCC) */
    #define v4l2_fourcc(a, b, c, d)\
      ((__u32)(a) | ((__u32)(b) << 8) | ((__u32)(c) << 16) | ((__u32)(d) << 24))


How is `pipe->modules` determined?
-------------

`modules` is an `|`d register including things like resizer and preview. In fact, I believe those are the only two modules to date.
