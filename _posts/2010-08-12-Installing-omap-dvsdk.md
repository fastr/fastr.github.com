---
layout: article
title: Installing omap dvsdk
categories: unfinished dvsdk
updated_at: 2010-08-12
---
Goal
====

Install dvsdk in /opt and get the examples running

Procedure
=========

    # download from ti
    ~/Downloads/dvsdk_3_01_00_10_Setup.bin

In the graphical installer that comes up:
    # select language
    # click Next
    # accept yes
    # destination folder: /opt/omap_dvsdk
    # finish

Environmental Variables to be Wary of
----

found in makefiles
  * DVSDK_INSTALL_DIR
  * CSTOOL_DIR
  * OMAP3503_SDK_INSTALL_DIR
  * LINUXKERNEL_INSTALL_DIR
  * UBOOT_INSTALL_DIR
  * CODEC_INSTALL_DIR
  * CODEGEN_INSTALL_DIR
  * EXEC_DIR

    cd /opt/omap_dvsdk/dvsdk_3_01_00_10
    git init
    git add .
    git commit -m "vanilla omap-dvsdk"

