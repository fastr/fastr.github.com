---
layout: article
title: Installing OMAP DVSDK on Gumstix Overo
categories: unfinished dvsdk bitbake
updated_at: 2010-07-26
---
Goal
====

Get the TI demos running on the gumstix.

TI's DVSDK & EVM packages
=======

According to TI's [Getting Started Guide: OMAP35x DVEVM Software Setup](http://processors.wiki.ti.com/index.php/GSG:_OMAP35x_DVEVM_Software_Setup#Installing_the_DVSDK_Software_.28DVSDK_version_3.01.00.09_onwards.29), you'll need the following packages:

  * [dvsdk_3_01_00_10_Setup.bin](http://software-dl.ti.com/dsps/dsps_public_sw/sdo_sb/targetcontent/dvsdk/DVSDK_3_00/latest/index_FDS.html) [local](/files/dvsdk_3_01_00_10_Setup.bin)
  * [TI-C6x-CGT-v6.1.12.bin](http://software-dl.ti.com/dsps/dsps_public_sw/sdo_sb/targetcontent/dvsdk/DVSDK_3_00/latest/index_FDS.html) [local](/files/TI-C6x-CGT-v6.1.12.bin)
  * [data_dvsdk_3_01_00_10.tar.gz (direct link)](http://software-dl.ti.com/dsps/dsps_public_sw/sdo_sb/targetcontent/dvsdk/DVSDK_3_00/latest/exports/data_dvsdk_3_01_00_10.tar.gz)
  * [cs1omap3530_setupLinux_1_01_00-prebuilt-dvsdk3.01.00.10.bin](http://software-dl.ti.com/dsps/dsps_public_sw/sdo_sb/targetcontent/dvsdk/DVSDK_3_00/latest/index_FDS.html) [local](/files/cs1omap3530_setupLinux_1_01_00-prebuilt-dvsdk3.01.00.10.bin)
  * [AM35x-OMAP35x-PSP-SDK-03.00.01.06.tgz](http://software-dl.ti.com/dsps/dsps_public_sw/psp/LinuxPSP/OMAP_03_00/03_00_01_06/index_FDS.html) [local](/files/AM35x-OMAP35x-PSP-SDK-03.00.01.06.tgz) - Documentation found in `AM35x-OMAP35x-PSP-SDK-##.##.##.##/docs/omap3530/UserGuide-##.##.##.##.pdf`, once extracted. This is similar to the Overo image.
  * [codesoucery_tools (direct link)](http://www.codesourcery.com/sgpp/lite/arm/portal/package4571/public/arm-none-linux-gnueabi/arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2) - This is TI's version of OpenEmbedded's `gcc-cross` which contains `arm-none-gnueabi-gcc` and friends. However, if you want to go all the way with TI's setup, it may be useful. [local](/files/arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2)

For the sake of trying things the TI way first, I might recommend. From those packages, we won't actually use `codesourcery_tools` since gumstix provides `gcc-cross`, which fulfills the same function. 

Installing
=======

Unpacking
-------

You'll need to unpackage the downloads in specific locations and also unpackage some of the sub-packages.
Assuming that you've downloaded all of the above into `~/Downloads`:

    cd ~/Downloads
    ./dvsdk_3_01_00_10_Setup.bin # installs to ~/dvsdk
    ./cs1omap3530_setupLinux_1_01_00-prebuilt-dvsdk3.01.00.10.bin # Installs to ~/cs1omap3530
    ln -s ~/cs1omap3530/cs1omap3530_1_01_00 ~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00
    sudo ./ti_cgt_c6000_6.1.12_setup_linux_x86.bin # Installs to /opt/TI/C6000CGT6.1.12
    export C6X_C_DIR=/opt/TI/C6000CGT6.1.12/include:/opt/TI/C6000CGT6.1.12/lib
    echo "export C6X_C_DIR=/opt/TI/C6000CGT6.1.12/include:/opt/TI/C6000CGT6.1.12/lib" >> ~/.bashrc
    tar xf data_dvsdk_3_01_00_10.tar.gz -C ~/dvsdk/dvsdk_3_01_00_10/clips/
    tar xf AM35x-OMAP35x-PSP-SDK-03.00.01.06.tar.gz -C ~/
    cd ~/AM35x-OMAP35x-PSP-SDK-03.00.01.06/src/kernel/
    tar xf linux-03.00.01.06.tar.gz
    cd ~/AM35x-OMAP35x-PSP-SDK-03.00.01.06/src/u-boot/
    tar xf u-boot-03.00.01.06.tar.gz
    cd ~/AM35x-OMAP35x-PSP-SDK-03.00.01.06/src/examples
    tar xf examples.tar.gz

Now that everything is installed you must configure `~/dvsdk/dvsdk_3_01_00_10/Rules.make`.
Here's an example of what the values that should probably be chaged:

    * `DVSDK_INSTALL_DIR=$(HOME)/dvsdk/dvsdk_3_01_00_10`
    * `CODEGEN_INSTALL_DIR=/opt/TI/C6000CGT6.1.12`
    * `OMAP3503_SDK_INSTALL_DIR=$(HOME)/AM35x-OMAP35x-PSP-SDK-03.00.01.06`
    * `CSTOOL_DIR=${OVEROTOP}/tmp/cross/armv7a`
    * `CSTOOL_PREFIX=$(CSTOOL_DIR)/bin/arm-angstrom-linux-gnueabi-`

Because some of the packages don't respect CSTOOL_PREFIX as they ought, also link the OpenEmbedded toolchain to `arm-none-linux-gnueabi-`

    cd ${OVEROTOP}/tmp/cross/armv7a/bin
    ls | cut -d'-' -f5-99 | while read COMP
    do
      ln -s arm-angstrom-linux-gnueabi-${COMP} arm-none-linux-gnueabi-${COMP} 
    done

To test that everything is installed correctly, try to compile all of the TI packages (probably takes 30 min+).
    
    cd ~/dvsdk/dvsdk_3_01_00_10
    make help
    make clobber # super clean
    make everything
    make

Using the overo-patched TI PSP Kernel
========

Now that everything is installed correctly enough to compile the vanilla TI demos, let's get the overo psp kernel


Get `linux-omap-psp` (or any omap-psp based-branch of the kernel) downloaded and patched for the overo.

    cd ${OVEROTOP}
    bitbake -c configure linux-omap-psp # Downloads and patches `git_arago-project.org.git.people.sriram.ti-psp-omap.git_a6bad4464f985fdd3bed72e1b82dcbfc004d7869.tar.gz`
    ls ${OVEROTOP}/tmp/work/overo-angstrom-linux-gnueabi/linux-omap-psp-2.6.32-r81+gitra6bad4464f985fdd3bed72e1b82dcbfc004d7869/git

Right now `/AM35x-OMAP35x-PSP-SDK-03.00.01.06/src/kernel/linux-03.00.01.06` is being used as the kernel dir in `Rules.make`.
Change that to the desired kernel.

    echo ${OVEROTOP}/tmp/work/overo-angstrom-linux-gnueabi/linux-omap-psp-2.6.32-r81+gitra6bad4464f985fdd3bed72e1b82dcbfc004d7869/git
    cd ~/dvsdk/dvsdk_3_01_00_10/
    vim Rules.make
    #LINUXKERNEL_INSTALL_DIR=$(HOME)/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/linux-omap-psp-2.6.32-r81+gitra6bad4464f985fdd3bed72e1b82dcbfc004d7869/git

Also, the `Makefile` is configured to build with the `omap3_evm_defconfig`, but we want the overo_defconfig

    cd ~/dvsdk/dvsdk_3_01_00_10/
    vim Makefile
    # edit near "ifeq ($(PLATFORM),omap3530)"
    # LINUXKERNEL_CONFIG=overo_defconfig

And try to compile the kernel now

    make linux_clean
    make linux
    
Running the demos
========

Basically looking for something that copies data in and out of the codec.... not finished


Appendix: Examining the TI Packages
========

dvsdk v3 (dvsdk_3_01_00_10_Setup.bin & data_dvsdk_3_01_00_10.tar.gz)
--------

**Unpacking**:

  1. run `./dvsdk_3_01_00_10_Setup.bin` and unpack as `~/dvsdk`
  2. run `tar xf data_dvsdk_3_01_00_10.tar.gz -C ~/dvsdk/dvsdk_3_01_00_10/clips`
  3. run `ls ~/dvsdk/dvsdk_3_01_00_10` to see that the results are as expected

This package contains these other packages:

  * bios_5_41_00_06
  * biosutils_1_02_02
  * ceutils_1_06
  * cg_xml_v2_12_00
  * codec_engine_2_25_02_11
  * dmai_2_05_00_12
  * dsplink_linux_1_65_00_02
  * dvsdk_demos_3_01_00_13
  * dvtb_4_20_05
  * edma3_lld_01_11_00_03
  * framework_components_2_25_01_05
  * linuxlibs_3_01
  * linuxutils_2_25_02_08
  * local_power_manager_linux_1_24_02_09
  * xdais_6_25_02_11
  * xdctools_3_16_01_27

It also contains these directories:

  * bin - check.sh info.sh
  * clips - the data files are in the package **`data_dvsdk_3_01_00_10.tar.gz`**
  * docs - lies, all lies (there aren't any docs there)
  * examples - has an example `loadmodules.sh`
  * kernel_binaries - pre-built binaries for the provided kernel
  * targetfs - for building with `rootfs`

And these files:

  * dvsdk_3_01_00_10_releasenotes.pdf
  * Makefile - `make help` to find out
  * Rules.make - **edit this file** to match your configuration. Example:
    * `DVSDK_INSTALL_DIR=$(HOME)/dvsdk/dvsdk_3_01_00_10`
    * `CODEGEN_INSTALL_DIR=/opt/TI/C6000CGT6.1.12`
    * `OMAP3503_SDK_INSTALL_DIR=$(HOME)/AM35x-OMAP35x-PSP-SDK-03.00.01.06`
    * `CSTOOL_DIR=${OVEROTOP}/tmp/cross/armv7a`
    * `CSTOOL_PREFIX=$(CSTOOL_DIR)/bin/arm-angstrom-linux-gnueabi-`
  * uninstall


cs1omap3530 v1 (cs1omap3530_setupLinux_1_01_00-prebuilt-dvsdk3.01.00.10.bin)
---------

**Unpacking**:

  1. run `./cs1omap3530_setupLinux_1_01_00-prebuilt-dvsdk3.01.00.10.bin` and unpack to `~/cs1omap3530/`
  2. run `mv ~/cs1omap3530/cs1omap3530_1_01_00 ~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00`
  3. run `ls ~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00` to see that the results are as expected

This package contains an example *codec server*

  * `~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00/`
    * config.bld
    * cs1omap3530_release_notes.htm
    * csRules.mak
    * Makefile
  * `~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00/packages/ti/sdo/codecs/`
    * aachedec
    * deinterlacer
    * g711dec
    * g711enc
    * h264dec
    * h264enc
    * jpegdec
    * jpegenc
    * mpeg2dec
    * mpeg4dec
    * mpeg4enc
  * `~/dvsdk/dvsdk_3_01_00_10/cs1omap3530_1_01_00/packages/ti/sdo/server/cs/`
    * bin
    * codec.cfg
    * docs
    * link.cmd
    * main.c
    * package
    * package.bld
    * package.mak
    * package.xdc
    * package.xs
    * server.cfg
    * server.tcf

TI-C6x-CGT v6
---------

**Unpacking**:

  1. run `./TI-C6x-CGT-v6.1.12.bin` and unpack as `/opt/TI/C6000CGT6.1.12`
  2. run `ls /opt/TI/C6000CGT6.1.12` to see that the results are as expected

This package contains the DSP Toolchain: C6000 CodeGen Tools

  * bin - abs6x, ap6x, asm6x, ci6x, clist6x, dem6x, embed6x, ilk6x, lnk6x, load6xexe, nm6x, opt6x, pprof6x, strip6x, acp6x, ar6x, cg6x, cl6x, cmp6x, dis6x, hex6x, libinfo6x, load6x, mk6x, ofd6x, pdd6x, spkc6x.dll, xref6x
  * DefectHistory.txt
  * include - access.h, cctype, cpp_inline_math.h, cstring, fastmath67x.h, hash_map, iostream.h, linkage.h, mathl.h, rope, stddef.h, string, valarray, xiosbase, xmemory algorithm, cerrno, cpy_tbl.h, ctime, file.h, hash_set, _isfuncdcl.h, list, memory, set, stdexcept, string.h, vector, xlocale, xstddef assert.h, cfloat, csetjmp, ctype.h, float.h, inttypes.h, _isfuncdef.h, locale, new, setjmp.h, stdint.h, strstream, wchar.h, xlocinfo, xstring bitset, ciso646, csignal, cwchar, _fmt_specifier.h, iomanip, iso646.h, locale.h, new.h, signal.h, stdio.h, strstream.h, wchar.hx, xlocinfo.h, xtree c60asm.i, climits, cstdarg, cwctype, fstream, iomanip.h, istream, _lock.h, numeric, slist, stdiostream.h, time.h, wctype.h, xlocmes, xutility c6x.h, clocale, cstddef, deque, fstream.h, ios, iterator, map, ostream, sstream, stdlib.h, typeinfo, xcomplex, xlocmon, xwcc.h cargs.h, cmath, cstdio, errno.h, functional, iosfwd, limits, mathf.h, pprof.h, stack, stl.h, unaccess.h, xdebug, xlocnum, ymath.h cassert, complex, cstdlib, exception, gsm.h, iostream, limits.h, math.h, queue, stdarg.h, streambuf, utility, xhash, xloctime, yvals.h
  * lib - fastmath67xe.lib, libc.a, rts6200_eh.lib, rts6400e_eh.lib, rts6400.lib, rts64pluse.lib, rts6700_eh.lib, rts6740e_eh.lib, rts6740.lib, rts67pluse.lib, fastmath67x.lib, lnk.cmd, rts6200e.lib, rts6400_eh.lib, rts64pluse_eh.lib, rts64plus.lib, rts6700e.lib, rts6740_eh.lib, rts67pluse_eh.lib, rts67plus.lib, fastmath67x.src, rts6200e_eh.lib, rts6200.lib, rts6400e.lib, rts64plus_eh.lib, rts6700e_eh.lib, rts6700.lib, rts6740e.lib, rts67plus_eh.lib, rtssrc.zip
  * LINKER_README.txt
  * man
  * README.txt
  * uninstall_cgt_c6000.bin


Scratchpad (unorganized, unfinished)
==========


    bitbake -c clean linux-omap-psp
    bitbake -c clean ti-dsplink
    bitbake -c clean ti-linuxutils
    bitbake -c clean ti-dvsdk-demos

    bitbake linux-omap-psp # Downloads `git_arago-project.org.git.people.sriram.ti-psp-omap.git_a6bad4464f985fdd3bed72e1b82dcbfc004d7869.tar.gz`
    bitbake ti-dsplink # Downloads `dsplink_linux_1_65_00_02.tar.gz`
    bitbake ti-linuxutils # Downloads `linuxutils_2_25_01_06.tar.gz`
    bitbake ti-dvsdk-demos

You should be able to build the kernel just fine, but you'll need extra files from TI.
I'm not sure which all of these are needed, but here are some links to files that will prove useful.

  * [OMAP35x Applications Processors](http://focus.ti.com/dsp/docs/dspcontent.tsp?contentId=53403)
  * [DVSDK for Linux development](http://www.ti.com/omap35xlinuxdvsdk)
  * [LINUXDVSDK-OMAP](http://focus.ti.com/docs/toolsw/folders/print/linuxdvsdk-omap.html)
  * [XDC packaged codecs](http://software-dl.ti.com/dsps/dsps_public_sw/codecs/OMAP35xx/index_FDS.html)
  * [AES codec](http://focus.ti.com/docs/toolsw/folders/print/c64xpluscrypto.html)
  * [OMAP3530 Application Processor](http://focus.ti.com/docs/prod/folders/print/omap3530.html)
  * [OMAP3530 Datasheet](http://www.ti.com/litv/pdf/sprs507f)
  * [OMAP DVSDK Getting Started](http://processors.wiki.ti.com/index.php/GSG:_OMAP35x_DVEVM_Software_Setup#Installing_the_Software)

Building DVSDK
======

  * [Rebuilding the DVSDK software for the target](http://processors.wiki.ti.com/index.php/GSG:_OMAP35x_DVEVM_Software_Setup#Rebuilding_the_DVSDK_software_for_the_target)

What we don't need
=======

  * MLO, boot_lcd.scr, boot_dvi.scr, sdimage_dvsdk_3_01_00_10.tar.gz - These are specific for the DVEVM board.

linux-omap-psp
==============

This should build without issue.

If you get confused as to which of the lovely files in `~/overo-oe/tmp/deploy/glibc/images/` is the kerenl you're looking for, try `ls -t` which will list the most recenly built kernels first.

without bitbake
---------------
  
  http://www.jumpnowtek.com/index.php?option=com_content&view=article&id=46&Itemid=54

    bitbake -c configure linux-omap-psp
    cd todo some directory
    make menuconfig

no, really without bitbake
--------------------------

Grab the kernel source straight from the TI guy:

    git clone git://arago-project.org/git/people/sriram/ti-psp-omap.git;protocol=git;branch=master
    
Set up a minimal set of env build variables

  kernel.env:

    CROSS_DIR=~/overo-oe/tmp/cross/armv7a/bin
    CROSS_PREFIX=arm-angstrom-linux-gnueabi-
    export ARCH=arm
    export CROSS_COMPILE=${CROSS_DIR}/${CROSS_PREFIX}

Configure the kernel

    source kernel.env
    make overo_defconfig
    make menuconfig

  kernel-dev.env:

    if [[ -z "${KERNEL_CROSS_BUILD_ENVIRONMENT_SOURCED}" ]]; then

      # should work for MACHINE=beagleboard also, you will have to set OETMP correctly
      MACHINE=overo

      # Set OETMP to be your OE config TMPDIR. This is a configurable param for OE. 

      # For gumstix setups, look in ${OVEROTOP}/build/conf/site.conf
      # This is the gumstix default.
      OETMP=${OVEROTOP}/tmp

      # For beagleboard setups, look in ${OETREE}/build/conf/local.conf
      # This is the beagleboard default.
      # OETMP=${OETREE}/tmp

      SYSROOTSDIR=${OETMP}/sysroots
      STAGEDIR=${SYSROOTSDIR}/`uname -m`-linux
      CROSSBINDIR=${OETMP}/cross/armv7a/bin

      export KERNELDIR=${SYSROOTSDIR}/${MACHINE}-angstrom-linux-gnueabi/kernel

      PATH=${STAGEDIR}/bin:${PATH}
      PATH=${STAGEDIR}/sbin:${PATH}
      PATH=${CROSSBINDIR}:${PATH}
      PATH=${STAGEDIR}/usr/bin:${PATH}
      PATH=${STAGEDIR}/usr/sbin:${PATH}
      PATH=${STAGEDIR}/usr/bin/armv7a-angstrom-linux-gnueabi:${PATH}

      unset CFLAGS CPPFLAGS CXXFLAGS LDFLAGS MACHINE

      export ARCH="arm"
      export CROSS_COMPILE="arm-angstrom-linux-gnueabi-"
      export CC="arm-angstrom-linux-gnueabi-gcc"
      export LD="arm-angstrom-linux-gnueabi-ld"
      export KERNEL_CROSS_BUILD_ENVIRONMENT_SOURCED="true"

      echo "Altered environment for cross building kernel/u-boot with OE tools."
    else
      echo "Cross build environment already configured."
    fi

To include that in the environment:

    source kernel-dev.env
    make overo_defconfig
    make menuconfig

ti-linuxutils
=============

cmemk.ko
--------

    insmod /lib/modules/2.6.32/kernel/drivers/dsp/cmemk.ko 
    CMEMK module: built on Jul 26 2010 at 17:53:26
      Reference Linux version 2.6.32
      File /home/user/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.c
    no physical memory specified, continuing with no memory allocation capability...
    cmemk initialized
    rmmod cmemk.ko


Child Packages
==============

A helpful FYI: Those two packages (ti-linuxutils, ti-dsplink) contain all of these:

    ./tmp/deploy/glibc/ipk/overo/
      kernel-firmware-ti-3410_2.6.33-r80.5_overo.ipk
      kernel-firmware-ti-5052_2.6.33-r80.5_overo.ipk
      kernel-module-ti-usb-3410-5052_2.6.33-r80.5_overo.ipk
      ti-biosutils-dbg_1_02_02-r1.5_overo.ipk
      ti-biosutils-dev_1_02_02-r1.5_overo.ipk
      ti-biosutils-sourcetree_1_02_02-r1.5_overo.ipk
      ti-biosutils_1_02_02-r1.5_overo.ipk
      ti-cgt6x-dbg_6_1_9-r4.5_overo.ipk
      ti-cgt6x-dev_6_1_9-r4.5_overo.ipk
      ti-cgt6x-sourcetree_6_1_9-r4.5_overo.ipk
      ti-cgt6x_6_1_9-r4.5_overo.ipk
      ti-codec-engine-dbg_2_25_01_06-r5.5_overo.ipk
      ti-codec-engine-dev_2_25_01_06-r5.5_overo.ipk
      ti-codec-engine-examples_2_25_01_06-r5.5_overo.ipk
      ti-codec-engine-sourcetree_2_25_01_06-r5.5_overo.ipk
      ti-codec-engine_2_25_01_06-r5.5_overo.ipk
      ti-dspbios-dbg_5_41_02_14-r1.5_overo.ipk
      ti-dspbios-dev_5_41_02_14-r1.5_overo.ipk
      ti-dspbios-sourcetree_5_41_02_14-r1.5_overo.ipk
      ti-dspbios_5_41_02_14-r1.5_overo.ipk
      ti-dsplink-dbg_1_64-r80f.5_overo.ipk
      ti-dsplink-dev_1_64-r80f.5_overo.ipk
      ti-dsplink-examples_1_64-r80f.5_overo.ipk
      ti-dsplink-module_1_64-r80f.5_overo.ipk
      ti-dsplink-sourcetree_1_64-r80f.5_overo.ipk
      ti-dsplink_1_64-r80f.5_overo.ipk
      ti-edma3lld-dbg_01_11_00_03-r0.5_overo.ipk
      ti-edma3lld-dev_01_11_00_03-r0.5_overo.ipk
      ti-edma3lld-sourcetree_01_11_00_03-r0.5_overo.ipk
      ti-edma3lld_01_11_00_03-r0.5_overo.ipk
      ti-framework-components-dbg_2_25_01_05-r1.5_overo.ipk
      ti-framework-components-dev_2_25_01_05-r1.5_overo.ipk
      ti-framework-components-sourcetree_2_25_01_05-r1.5_overo.ipk
      ti-framework-components_2_25_01_05-r1.5_overo.ipk
      ti-local-power-manager-dbg_1_24_01-r80d.5_overo.ipk
      ti-local-power-manager-dev_1_24_01-r80d.5_overo.ipk
      ti-local-power-manager-sourcetree_1_24_01-r80d.5_overo.ipk
      ti-local-power-manager_1_24_01-r80d.5_overo.ipk
      ti-lpm-module_1_24_01-r80d.5_overo.ipk
      ti-lpm-utils_1_24_01-r80d.5_overo.ipk
      ti-xdais-dbg_6_25_01_08-r1.5_overo.ipk
      ti-xdais-dev_6_25_01_08-r1.5_overo.ipk
      ti-xdais-sourcetree_6_25_01_08-r1.5_overo.ipk
      ti-xdais_6_25_01_08-r1.5_overo.ipk
      ti-xdctools-dbg_3_16_01_27-r2.5_overo.ipk
      ti-xdctools-dev_3_16_01_27-r2.5_overo.ipk
      ti-xdctools-sourcetree_3_16_01_27-r2.5_overo.ipk
      ti-xdctools_3_16_01_27-r2.5_overo.ipk

The individual bitbake files are these:

    find ${OVEROTOP}/org.openembedded.dev/recipes/ | grep '\<ti\>' | grep bb | cut -d'/' -f3-99 | cut -d'_' -f1 | sort -u
    org.openembedded.dev/recipes/
      firmwares/firmware-ti-wl1251.bb
      images/ti-codec-engine-test-image.bb
      images/ti-demo-x11-image.bb
      julius/ti-julius-demo
      tasks/task-gstreamer-ti.bb
      ti/am-benchmarks
      ti/am-sysinfo
      ti/bitblit
      ti/gstreamer-ti
      ti/matrix-gui
      ti/matrix-gui-common
      ti/matrix-gui-e
      ti/matrix-tui
      ti/ti-audio-soc-example
      ti/ti-biospsp
      ti/ti-biosutils
      ti/ti-cgt6x
      ti/ti-codec-engine
      ti/ti-codecs-dm355
      ti/ti-codecs-dm365
      ti/ti-codecs-dm6446
      ti/ti-codecs-dm6467
      ti/ti-codecs-omap3530
      ti/ti-codecs-omapl137
      ti/ti-codecs-omapl138
      ti/ti-devshell.bb
      ti/ti-dm355mm-module
      ti/ti-dm365mm-module
      ti/ti-dmai
      ti/ti-dspbios
      ti/ti-dsplib
      ti/ti-dsplink
      ti/ti-dvsdk-demos
      ti/ti-edma3lld
      ti/ti-framework-components
      ti/ti-linuxutils
      ti/ti-local-power-manager
      ti/ti-msp430-chronos
      ti/ti-sysbios
      ti/ti-xdais
      ti/ti-xdctools

Errors
======

If you try to compile from the default overo kernel (or your haven't cleaned your previous attempt of ti-xzy) you're likely to run into these errors:

    bitbake ti-linuxutils
      |   CC [M]  /home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.o
      | /home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.c:65:2: warning: #warning *** not a warning *** Note: LINUX_VERSION_CODE >= 2.6.26
      | /home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.c: In function 'ioctl':
      | /home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.c:1530: error: implicit declaration of function 'dmac_clean_range'
      | /home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.c:1541: error: implicit declaration of function 'dmac_inv_range'
      | make[4]: *** [/home/coolaj86/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/ti-linuxutils-1_2_25_01_06-r80c/linuxutils_2_25_01_06/packages/ti/sdo/linuxutils/cmem/src/module/cmemk.o] Error 1


According to the [e2e forum](http://e2e.ti.com/support/dsp/omap_applications_processors/f/447/p/46875/165433.aspx) it's possible to [patch](http://git.igep.es/?p=pub/scm/linux-omap-2.6.git;a=commit;h=ccbcd5d0a831a406dc01ba85f014fb8443ee79b5) the kernel (essentially adding a `http://` entry to the `linux-omap3_2.6.33.bb` (which has been removed, btw).
The question remains: what replaces dmac_clean_range? and wouldn't it be better to patch cmemk with the update?, but judging by linux-ti-omap4 I'm guessing that the revert patch is pretty popular and perhaps the removal was premature.

