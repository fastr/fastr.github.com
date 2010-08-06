---
layout: article
title: ti dsplink on OpenEmbedded
categories: uncategorized
updated_at: 2010-07-26
---

Disclaimer: I'm using a Gumstix Overo, which uses Angstrom. I assume that this will be helpful to most anyone developing on OpenEmbedded, however, YMMV.

If you've been developing not realizing what git is, now would be a good time to update your repository with `git fetch` because you'll need the linux-omap-psp kernel that was just recently merged into the mainstream overo branch.

    ~/overo-oe/org.openembedded.dev
    git fetch origin
    git pull origin overo

Next you can make a feedble attempt to instal the bitbake packages you'll need.

    bitbake -c clean linux-omap-psp
    bitbake -c clean ti-dsplink
    bitbake -c clean ti-linuxutils
    
    bitbake linux-omap-psp
    bitbake ti-dsplink
    bitbake ti-linuxutils

You should be able to build the kernel just fine, but you'll need extra files from TI.
I'm not sure which all of these are needed, but here are some links to files that will prove useful.

  * [OMAP35x Applications Processors](http://focus.ti.com/dsp/docs/dspcontent.tsp?contentId=53403)
  * [DVSDK for Linux development](http://www.ti.com/omap35xlinuxdvsdk)
  * [LINUXDVSDK-OMAP](http://focus.ti.com/docs/toolsw/folders/print/linuxdvsdk-omap.html)
  * [XDC packaged codecs](http://software-dl.ti.com/dsps/dsps_public_sw/codecs/OMAP35xx/index_FDS.html)
  * [AES codec](http://focus.ti.com/docs/toolsw/folders/print/c64xpluscrypto.html)
  * [OMAP3530 Application Processor](http://focus.ti.com/docs/prod/folders/print/omap3530.html)
  * [OMAP3530 Datasheet](http://www.ti.com/litv/pdf/sprs507f)


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

A helpful FYI: Those two packages contain all of these:

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

