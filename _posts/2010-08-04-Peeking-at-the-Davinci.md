---
layout: article
title: Peeking at the Davinci
categories: isp-ccdc davinci
updated_at: 2010-08-04
---

Goal
====

Find the linux kernel source for the old TMS320DM6446 (DM64x) Davinci.

There is a similar driver, the mt9t001, which may provide some insight into creating a custom raw capture driver.

Finding the Kernel
==================

It looks like public access is not possible and or that the extranet site on which it was hosted no longer exists.

If you can find it elsewhere, the file is likely called something like `DaVinciLSP-REL_mvl401c.tar.gz` or `DaVinciLSP-REL_mvl401i.tar.gz`

Resources
=========

  * [Getting started with DaVinci evaluation kit (DVEVM6446) SW](http://web.sysart.fi/developer/linux:architectures:arm:davinci:getting_started_evmdm6446)

Appendix
========

While trying to find the kernel, I first ended up finding the `davinci_dvsdk` instead:

  1. Visit ti.com
  2. Search 'davinci' or 'dm64x'
  3. Navigate to [TMS320DM6446](http://focus.ti.com/docs/prod/folders/print/tms320dm6446.html)
  4. Find [LINUXDVSDK-DV](http://focus.ti.com/docs/toolsw/folders/print/linuxdvsdk-dv.html)

The old way to get to the files was something like:

  1. visit ti.com
  2. click my.TI Login
  3. enter credentials
  4. click Extranets
  5. Search DaVinci
  6. DM644x
  7. Version 1.30 / 2.0

