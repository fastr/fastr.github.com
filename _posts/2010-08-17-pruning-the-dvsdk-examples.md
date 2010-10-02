---
layout: article
uuid: 06331941-0c74-4f1f-94f1-5448c597fb49
title: pruning the dvsdk examples
categories: unfinished dvsdk
updated_at: 2010-08-17
---
Goal
====

Get down to a bite-size chunk of TI examples from the myriad of examples present.

Procedure
=========

Compiling the TI examples outside of `~/dvsdk`
---------

See the my prior post *"OMAP DVSDK on OpenEmbedded"* in the dvsdk category for getting working the EVM examples working on the Gumstix Overo.

    cp -a ~/dvsdk/dvsdk_3_01_00_10/codec_engine_2_25_02_11/examples/ ~/ti-examples
    cd ~/ti-examples
    vim xdcpaths.mak

Here's an example using the changes I made:

    diff --git examples/xdcpaths.mak examples-pruned/xdcpaths.mak
    index 38b5fac..9eb3a8c 100644
    --- examples/xdcpaths.mak
    +++ examples-pruned/xdcpaths.mak
    @@ -1,4 +1,4 @@
    -#
    +
     #  Copyright (c) 2010, Texas Instruments Incorporated
     #  All rights reserved.
     #
    @@ -58,7 +58,7 @@
     # indicating an "ARM-only codec" application.  The CE example Servers cannot be
     # built for DM357, and since setting PROGRAMS to APP_CLIENT introduces a
     # dependency on the Server, we can't support APP_CLIENT on DM357.
    -DEVICES  := DM6446 DM6467 DM6437 DM355 DM365 OMAP2530 OMAP3530 OMAPL137 OMAPL138 X86
    +DEVICES  := OMAP3530 X86
     
     # (Optional) remove from this list the GPP OS's you're not interested in
     # building.  In most cases, you'll likely only leave one, as WinCE users don't
    @@ -66,7 +66,7 @@ DEVICES  := DM6446 DM6467 DM6437 DM355 DM365 OMAP2530 OMAP3530 OMAPL137 OMAPL138
     # GCC or uClibc, but not both)
     #
     # Note, this is a space-delimited list.
    -GPPOS := LINUX_GCC LINUX_UCLIBC WINCE
    +GPPOS := LINUX_GCC
     
     # (Optional) Remove from the list the types of programs you're not
     # interested in building:
    @@ -76,7 +76,7 @@ GPPOS := LINUX_GCC LINUX_UCLIBC WINCE
     # APP_LOCAL  -- Client+codecs in a single program, whether ARM only or DSP only
     #
     # Note, this is a space-delimited list.
    -PROGRAMS := APP_CLIENT DSP_SERVER APP_LOCAL
    +PROGRAMS := APP_CLIENT DSP_SERVER
     
     # (Mandatory) Specify where various components are installed.
     # What you need depends on what device(s) you're building for, what type(s) of
    @@ -97,10 +97,12 @@ PROGRAMS := APP_CLIENT DSP_SERVER APP_LOCAL
     # need (will be warned if you do, based on your DEVICES + PROGRAMS selection
     # above).
    - 
    -CE_INSTALL_DIR        := _your_CE_installation_directory_/codec_engine_2_25_02_11
    -XDC_INSTALL_DIR       := _your_XDCTOOLS_installation_directory/xdctools_3_16_00_18
    -BIOS_INSTALL_DIR      := _your_SABIOS_installation_directory/bios_5_41_00_06
    -DSPLINK_INSTALL_DIR   := _your_DSPLink_installation_directory/dsplink-1_61_03-prebuilt
    +
    +DVSDK_INSTALL_DIR     := $(HOME)/dvsdk/dvsdk_3_01_00_10
    +
    +CE_INSTALL_DIR        := $(DVSDK_INSTALL_DIR)/codec_engine_2_25_02_11
    +XDC_INSTALL_DIR       := $(DVSDK_INSTALL_DIR)/xdctools_3_16_00_18
    +BIOS_INSTALL_DIR      := $(DVSDK_INSTALL_DIR)/bios_5_41_00_06
    +DSPLINK_INSTALL_DIR   := $(DVSDK_INSTALL_DIR)/dsplink_linux_1_65_00_02/packages
     
     # no need to specify the installation directories below if your CE installation
     # has cetools/ directory on top
    @@ -108,13 +110,13 @@ DSPLINK_INSTALL_DIR   := _your_DSPLink_installation_directory/dsplink-1_61_03-pr
     # Note, CMEM_INSTALL_DIR is a misnomer - it should be LINUXUTILS_INSTALL_DIR
     # but we've got an existing user base.  Need to fix this later.
     USE_CETOOLS_IF_EXISTS := 1
    -XDAIS_INSTALL_DIR     := _your_xDAIS_installation_directory/xdais_6_25_02_11
    -FC_INSTALL_DIR        := _your_FC_installation_directory/framework_components_2_25_01_05
    -CMEM_INSTALL_DIR      := _your_CMEM_installation_directory/linuxutils_2_25_02_08
    -WINCEUTILS_INSTALL_DIR:= _your_WINCEUTILS_installation_directory/winceutils_1_00_03_13
    -BIOSUTILS_INSTALL_DIR := _your_BIOSUTILS_installation_directory/biosutils_1_02_02
    -EDMA3_LLD_INSTALL_DIR := _your_EDMA3_LLD_installation_directory/edma3_lld_01_11_00_02
    -LPM_INSTALL_DIR       := _your_LPM_installation_directory/local_power_manager_1_24_02_09
    +XDAIS_INSTALL_DIR     := $(DVSDK_INSTALL_DIR)/xdais_6_25_02_11
    +FC_INSTALL_DIR        := $(DVSDK_INSTALL_DIR)/framework_components_2_25_01_05
    +CMEM_INSTALL_DIR      := $(DVSDK_INSTALL_DIR)/linuxutils_2_25_02_08
    +WINCEUTILS_INSTALL_DIR:= $(DVSDK_INSTALL_DIR)/winceutils_1_00_03_13
    +BIOSUTILS_INSTALL_DIR := $(DVSDK_INSTALL_DIR)/biosutils_1_02_02
    +EDMA3_LLD_INSTALL_DIR := $(DVSDK_INSTALL_DIR)/edma3_lld_01_11_00_02
    +LPM_INSTALL_DIR       := $(DVSDK_INSTALL_DIR)/local_power_manager_1_24_02_09
     
     
     # (Mandatory) specify correct compiler paths and names for the architectures
    @@ -122,9 +124,10 @@ LPM_INSTALL_DIR       := _your_LPM_installation_directory/local_power_manager_1_
     
     # compiler path and name for Montavista Arm 9 toolchain. NOTE: make sure the
     # directory you specify has a "bin" subdirectory
    -CGTOOLS_V5T := /db/toolsrc/library/tools/vendors/cs/arm/arm-2007q3
    -CC_V5T := bin/arm-none-linux-gnueabi-gcc
    -CGTARGET := gnu.targets.arm.GCArmv5T
    +CGTOOLS_V5T := $(HOME)/overo-oe/tmp/cross/armv7a
    +CC_V5T := bin/arm-angstrom-linux-gnueabi-gcc
    +CGTARGET := gnu.targets.arm.GCArmv5T
    +# TODO can we use GCArmv7A?
     
     # compiler path and name for uClibc toolchain. NOTE: make sure the
     # directory you specify has a "bin" subdirectory
    @@ -137,17 +140,17 @@ WINCE_PROJECTROOT := $(WINCE_ROOTDIR)/_your_ProjectRoot_/Wince600/TI_EVM_3530_AR
     
     # compiler path and name for TI C64x toolchain. NOTE: make sure the
     # directory you specify has a "bin" subdirectory
    -CGTOOLS_C64P := /db/toolsrc/library/tools/vendors/ti/c6x/6.0.16/Linux
    +CGTOOLS_C64P := /opt/TI/C6000CGT6.1.12
     #CC_C64P      := bin/cl6x
     
     # compiler path and name for TI C674 toolchain. NOTE: make sure the
     # directory you specify has a "bin" subdirectory
    -CGTOOLS_C674 := /db/toolsrc/library/tools/vendors/ti/c6x/6.1.5/Linux
    +CGTOOLS_C674 := /opt/TI/C6000CGT6.1.12
     #CC_C674      := bin/cl6x
     
     # compiler path and name for Linux 86 toolchain. NOTE: make sure the
     # directory you specify has a "bin" subdirectory
    -CGTOOLS_LINUX86 := _your_Linux86_installation_directory
    +CGTOOLS_LINUX86 := /usr
     #CC_LINUX86   := bin/gcc
     
     # -----------------------------------------------------------------------------

I tested that I could `make clean && make` and it worked. Amazaing!

Compiling fewer of the TI examples
------------------

    cp -a ~/ti-examples ~/ti-examples-pruned
    vim ~/ti-examples-pruned/makefile

The interesting parts of the top-level `examples` folder are buried 4 levels deep.

  * `./ti/sdo/ce/examples`
    * `apps`
    * `codecs`
    * `servers`

I decided (rather arbitrarily) that I'm just interested in `video_copy` and `all_codecs_new_config`, so I deleted everything not related and kept making adjustments to `makefile`s, `package.xdc`s,  and `all_codecs_new_config/all.cfg` until it compiled.

I also got rid of

  * all of the platforms in `./apps/video_copy/configuro/` and `.tci`s excepting the `evm3530`, which is closest to the gumstix.
  * all of the `package` folders

I may go back and try to get `audio_copy` and `server_api_example` as well.

Creating my own package
-----------

After trimming the fat from the file system I decided to lighten the directory structure as well and work towards getting my own project built.

    mkdir -p ./faster-examples
    cp -a ti-examples-pruned/ti/sdo/ce/examples/ ./fastr-examples/fastr_ce
    cp ti-examples-pruned/makefile ./fastr-examples/
    cp ti-examples-pruned/xdcpaths.mak ./fastr-examples/
    cp ti-examples-pruned/config.bld ./fastr-examples/
    cp ti-examples-pruned/buildutils/xdcrules.mak ./fastr-examples-extracted/buildutils/
    OLD='ti.sdo.ce.examples'
    NEW='fastr_ce'
    find ./ -type f | grep -v '.svn' | xargs sed -i "s/${OLD}/${NEW}/gi"

And I made a makefile for the top-level:

`./faster-examples/makefile`

    all:
    %::
      $(MAKE) -C fastr_ce $@

`./faster-examples/makefile`

    all:
    %::
      $(MAKE) -C codecs $@
      $(MAKE) -C servers $@
      $(MAKE) -C apps $@
    
Again, I had to muck around a bit with the `makefile`s - particularly having the right number of parent paths `../../..`.

Creating my own Codec Engine app
---------------

    cd ~/fastr-examples
    OLD='all_codecs_new_config'
    NEW='fsr_codecs'
    find ./ -type f | grep -v '.svn' | xargs sed -i "s/${OLD}/${NEW}/gi"
    mv fastr_ce/all_codecs_new_config fastr_ce/fsr_codecs

    cd ~/fastr-examples
    OLD='all_codecs_new_config'
    NEW='fastr_codecs'
    find ./ -type f | grep -v '.svn' | xargs sed -i "s/${OLD}/${NEW}/gi"
    cd ~/fastr-examples/fastr_ce/apps
    mv videnc_copy fodec

    vim makefile # add `fodec` - the fastr codec
    
