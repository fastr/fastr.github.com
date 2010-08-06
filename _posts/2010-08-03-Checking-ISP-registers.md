---
layout: article
title: Checking ISP registers
categories: uncategorized
updated_at: 2010-08-03
---

Goal
====

Read, set, and check ISP registers

Use a raw capture from the CCDC to put the data into memory, bypassing the preview, resizer, and other ISP auxilaries.

Status
------

This document is just a note-taking place right now. Don't expect useful information below.

Resources
=========

  * [OMAP35x Peripherals Overview Reference Guide](http://focus.ti.com/general/docs/lit/getliterature.tsp?literatureNumber=sprufn0a&fileType=pdf)
  * [OMAP35x Technical Reference Manual](http://www.ti.com/lit/pdf/spruf98)

Glossary
========

  * [CCD](http://en.wikipedia.org/wiki/Charge-coupled_device) - charge-coupled device - image sensor
  * CCDC - CCD Camera
  * ISP - Image Signal Processor
  * SBL - Shared Logic Buffer
  * PRCM - ?

Outline of ISP documentation in SPRUF98G
============================

Since TI didn't have the decency to create HTML documentation, nor to sort the documentation by topic, here's a bite-size breakdown of ISP-related information in spruf98g.pdf:

Contents:

  * 00.0 Camera ISP Index .................................................................................................... 21
  * 12.1 Camera ISP Overview ................................................................................................. 1308
  * 12.2 Camera ISP Environment ............................................................................................. 1313
  * 12.3 Camera ISP Integration ................................................................................................ 1359
  * 12.4 Camera ISP Functional Description .................................................................................. 1370
  * 12.5 Camera ISP Basic Programming Model ............................................................................. 1443
  * 12.6 Camera ISP Register Manual ......................................................................................... 1494

Figures:

  * 12-53. **Camera ISP Block Diagram**........................................................................................... 1371
  * 12-54. Camera ISP/Data Path/RAW RGB Images ......................................................................... 1373
  * 12-55. Camera ISP/Data Path/YUV4:2:2 Images .......................................................................... 1374
  * 12-56. Camera ISP/Data Path/JPEG Images ............................................................................... 1374

Tables:

  * 12-1. Camera ISP Functions................................................................................................. 1313
  * 12-17. Camera ISP Interrupts ................................................................................................. 1364
  * 12-68. Camera ISP Instance Summary...................................................................................... 1494
  * 12-69. ISP Register Summary ................................................................................................ 1494
  * 12-70. ISP_REVISION ......................................................................................................... 1495
  * 12-71. Register Call Summary for Register ISP_REVISION.............................................................. 1495
  * 12-72. ISP_SYSCONFIG ...................................................................................................... 1495
  * 12-73. Register Call Summary for Register ISP_SYSCONFIG........................................................... 1496
  * 12-74. ISP_SYSSTATUS ...................................................................................................... 1496
  * 12-75. Register Call Summary for Register ISP_SYSSTATUS........................................................... 1496
  * 12-76. ISP_IRQ0ENABLE ..................................................................................................... 1497
  * 12-77. Register Call Summary for Register ISP_IRQ0ENABLE.......................................................... 1499
  * 12-78. ISP_IRQ0STATUS ..................................................................................................... 1500
  * 12-79. Register Call Summary for Register ISP_IRQ0STATUS.......................................................... 1503
  * 12-80. ISP_IRQ1ENABLE ..................................................................................................... 1503
  * 12-81. Register Call Summary for Register ISP_IRQ1ENABLE.......................................................... 1506
  * 12-82. ISP_IRQ1STATUS ..................................................................................................... 1506
  * 12-83. Register Call Summary for Register ISP_IRQ1STATUS.......................................................... 1509
  * 12-88. ISP_CTRL ............................................................................................................... 1511
  * 12-89. Register Call Summary for Register ISP_CTRL.................................................................... 1514
  * 12-106. ISP_CBUFF Register Summary..................................................................................... 1521
  * 12-129. ISP_CSI1B Register Summary...................................................................................... 1529
  * 12-188. ISP_CCDC Register Summary...................................................................................... 1549
  * 12-259. ISP_HIST Register Summary ....................................................................................... 1581
  * 12-282. ISP_H3A Register Summary ........................................................................................ 1588
  * 12-331. ISP_PREVIEW Register Summary ................................................................................. 1603
  * 12-402. ISP_RESIZER Register Summary.................................................................................. 1628
  * 12-489. ISP_SBL Register Summary ........................................................................................ 1653
  * 12-616. ISP_CSI2A Register Summary...................................................................................... 1699

Outline of SPRS507F - OMAP3530/25 Applications Processor
===============

  * 2.5.2 Video Interfaces - page 98-100

Relating to the Linux Kernel
============================

  * 12.6 Camera ISP Register Manual ......................................................................................... 1494



  * disable `CSI1` with `CSI1_CTRL [0] IF_EN = 0x0`
  * disable `CSI2` with `CM_FCLKEN_CAM[1] EN_CSI2 = 0`

12.5.3 Programming the Timing CTRL Module

12.5.4 Programming the CCDC

Table 12-50. CCDC Required Configuration Parameters

  * `ISP_CTRL[3:2] PAR_BRIDGE`
  * `ISP_CTRL[7:6] SHIFT`
  * `ISP_CTRL[4] PAR_CLK_POL`

12.5.4.6.1.1 Input-Mode Selection
=======

SYNC mode:
----------

  - In this mode, the input data can be either raw data or YCbCr data. Setting
**`CCDC_SYN_MODE.INPMODE = 0` selects raw data**, and `CCDC_SYN_MODE[13:12] INPMODE = 1` or `2` 
selects YCbCr data on 16 or 8 bits. **If `CCDC_SYN_MODE[13:12] INPMODE = 0`, the cam_d
signal width is selected through `CCDC_SYN_MODE[10:8] DATSIZ`: the possible values are 8, 10,
11, and 12 bits.**
  - If `CCDC_SYN_MODE[13:12] INPMODE = 1`, the cam_d signal width is 8 bits, but the internal
CCDC module data path is configured to 16 bits. It is mandatory to enable the 8- to 16-bit bridge by
setting `ISP_CTRL[3:2] PAR_BRIDGE = 2` or `3`. The `ISP_CTRL [3:2] PAR_BRIDGE` bit also controls
how the 8-bit data is mapped onto the 16-bit data.
  - The value set in `CCDC_SYN_MODE[10:8] DATSIZ` does not matter. The position of the Y
component can be set with the `CCDC_CFG[11] Y8POS` bit.
  - If `CCDC_SYN_MODE[13:12] INPMODE = 2`, the cam_d signal width is 8 bits. The value set in
`CCDC_SYN_MODE[10:8] DATSIZ` does not matter. The position of the Y component can be set
with the `CCDC_CFG[11] Y8POS` bit.
  - **The internal timing generator must be enabled with `CCDC_SYN_MODE[16] VDHDEN = 1`.**

NOTE:

  - **`CCDC_REC656IF[0] R656ON = 0`** to disable ITU mode


12.5.4.6.1.2 Timing Generator and Frame Settings
=====

The polarities of the `cam_hs`, `cam_vs`, and `cam_fld` signals are controlled by the `CCDC_SYN_MODE[3]
HDPOL`, `CCDC_SYN_MODE[2] VDPOL`, and `CCDC_SYN_MODE[4] FLDPOL` bit fields. The polarities can
be positive or negative.

The pixel data is presented on cam_d one pixel for every `cam_pclk` rising edge or falling edge. It is
controlled with the `ISP_CTRL[4] PAR_CLK_POL` bit.

The `CCDC_SYN_MODE[7] FLDMODE` bit fields set the image-sensor type to progressive or interlaced
mode. When the sensor is interlaced, the `CCDC_SYN_MODE[15] FLDSTAT` status bit indicates whether
the current frame is odd or even.

The polarity of the cam_d signal can also be controlled with the `CCDC_SYN_MODE[6] DATAPOL` bit field.

The polarity can be normal mode or ones complement mode.

Furthermore, the directions of the `cam_fld` and `cam_hs`/`cam_vs` signals are controlled by the
`CCDC_SYN_MODE[1] FLDOUT` and `CCDC_SYN_MODE[0] VDHDOUT` bits. If `CCDC_SYN_MODE[0] VDHDOUT` 
is set as an output, the `CCDC_PIX_LINES` register controls the length of the `cam_hs` and
`cam_vs` signals.

If `CCDC_SYN_MODE[0] VDHDOUT = 1`:

  - The HS sync pulse width is given by `CCDC_HD_VD_WID[27:16] HDW`. The VS sync pulse width is
given by `CCDC_HD_VD [11:0] VDW`.
  - The HS period is given by `CCDC_PIX_LINES[31:16] PPLN`. The VS period is given by
`CCDC_PIX_LINES[15:0] HLPRF x 2`.

Figure 12-107 shows the HS/VS sync pulse output timings.

12.5.4.6.2 Image-Signal Processing
====

  - `CCDC_CLAMP[31] CLAMPEN = 0`
  - `CCDC_DCSUB = 0`

12.5.9 Programming the Central-Resource SBL
=====

???

Table 12-88. `ISP_CTRL`
======

  * `CCDC_RAM_EN`
  * `CCDC_CLK_EN`

12.6.5 Camera `ISP_CCDC` Registers
=====

  * `CCDC_SYN_MODE.VDHDOUT = 1` - `cam_hs` and `cam_vs` are output
  * `VDHDEN = 1` - enable timing generator
  * `INPMOD = 0` - Raw data
  * `HDW = 0` - how many pulses to leave hs sync (always at least 1)
  * `VDW = 0` - how many pulses to leave vs sync (always at least 1)

Table 12-193. `CCDC_SYN_MODE`
---




other
=====

`cam_pclk` must start before sending `cam_d` and start `cam_vs` and `cam_hs`

RAW can be processed via IVA2.2 in software

>  SYNC mode: In this mode, the `cam_hs` and `cam_vs` signals use dedicated wires.
>  Synchronization signals are provided by either the sensor or the camera ISP. This mode works
>  with 8-, 10-, 11-, and 12-bit data. It supports both progressive and interlaced image-sensor
>  modules.

12.4.6.1.3.1 SYNC CTRL Module
----------

The SYNC CTRL module receives the pixel-clock signal from the image sensor (PCLK). The module can
be slave or master of the horizontal and vertical synchronization signals (HS and VS) and of the
field-identification signal (FIELD).

The HS, VS, and FLD signals can be set as inputs or outputs. The polarity of the HS, VS, and FLD signals
can be set as positive or negative. If the HS, VS, and FLD signals are output, the signal length can be set.

For RAW data:
------------

  - Data is clipped to the number of LSBs specified in the `CCDC_SYN_MODE [10:8] DATSIZ` field.
This also sets the maximum data size allowed in subsequent clipping/limiting operations and is the
output data alignment if data is written to memory.

Conversion Area Select Parameters
--------

When the data formatter is enabled, HS/VS signals are still generated as output (`CCDC_SYN_MODE [16] VDHDEN = 0x1`).
The settings for these output signals are in the following fields:

  - `CCDC_HD_VD_WID[27:16] HDW`
  - `CCDC_HD_VD_WID[11:0] VDW`
  - `CCDC_PIX_LINES[31:16] PPLN`
  - `CCDC_PIX_LINES[15:0] HLPRF`

NOTE: These four registers are not used when HS/VS signals are input signals
(`CCDC_SYN_MODE [16] VDHDEN = 0x0`).

NOTE: The settings reflect those for the sensor readout frame, not the resultant reformatted frame.
Registers `CCDC_FMT_HORZ` and `CCDC_FMT_VERT` control the interpretation of the input data frame
when the data formatter is enabled.

Registers `CCDC_HORZ_INFO`, `CCDC_VERT_START`, and `CCDC_VERT_LINES` control the
interpretation of the input data frame in normal mode (when the data formatter is not enabled).


  * `CSI 0x480B C400 - 0x480B C5EC`
  * `CCDC 0x480B C600 - 0x480B C6A7`

`Resizer Physical Address 0x480B D000`

spruf98g.pdf - page 1636 - 

Table 12-423. `RSZ_HFILT10`

Description HORIZONTAL FILTER COEFFICIENTS 0 AND 1 REGISTER RW
Type RW

Address Offset 0x0000 0028 
Physical Address 0x480B D028

Instance  ISP_RESIZER 




On page 1458 there is a complete table with all the registers and values for those registers that need to be initialized for the CCDC to function.

Via the kernel
==============

It seems that `isp_interface_config` is the struct which is used to set the isp registers in the kernel

A Note to TI
============

Next time your Table of Contents is 163 pages, consider putting it up in o googleable format, maybe?
Or, better yet, don't do silly things - like create 34-hundred page manuals! Perhaps better to break it down by use-case or topic? Each major section in another manual unto itself?
