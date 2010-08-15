---
layout: article
title: CCDC hs vs as output via u-boot
categories: u-boot isp-ccdc
updated_at: 2010-08-12
---
Goal
====

Configure the `cam_hs` and `cam_vs` as outputs.

Linux Kernel
============

I had tried this before and documented the experience somewhat as [Export-gpio-pins-on-gumstix](http://fastr.github.com/2010-08-03/Export-gpio-pins-on-gumstix.html)

Later I found that although the `cam_d` lines were outputting data, the values were coming out as `0`s. There is a suspision that setting `CONFIG_OMAP3_MUX` in the kernel enables a number of other changes that may not be desired. The next step is to try via `node-devreg` and then `u-boot`.

    /* Camera - HS/VS signals */
    OMAP3_MUX(CAM_HS, OMAP_MUX_MODE0 | OMAP_PIN_INPUT),

    gpio94 / cam_hs

u-boot
=====

Make the changes you need:

    cd ${OVEROTOP}
    bitbake -c clean u-boot-omap3; bitbake -c configure u-boot-omap3
    VERSION=`bitbake --show-versions | grep 'u-boot-omap3\>' | cut -d':' -f2`
    UBOOTDIR=${OVEROTOP}/tmp/work/overo-angstrom-linux-gnueabi/u-boot-omap3-${VERSION}
    cd ${UBOOTDIR}/git/board/overo
    cp overo.h overo.h.orig
    vim overo.h # find `CAM_HS` and `CAM_VS` and change IEN (input enable) to IDIS (input disable)

Make your own copy of u-boot (as not to wipe it out if you clean org.openembedded.dev):

    cd ${OVEROTOP}/
    mkdir -p ./user.collection/recipes/
    cp -a ./org.openembedded.dev/recipes/u-boot ./user.collection/recipes/u-boot

Create a patchfile that bitbake can understand:

    cd ${UBOOTDIR}
    git diff --no-prefix git/board/overo/overo.h.orig git/board/overo/overo.h > pin-mux.patch
    mv pin-mux.patch ${OVEROTOP}/user.collection/recipes/u-boot/u-boot-omap3-git/

Edit the bitbake file:

    vim ${OVEROTOP}/user.collection/recipes/u-boot/u-boot-omap3_git.bb

So that it has `pin-mux.patch;patch=yes \`, something like this:

    SRC_URI = "git://www.sakoman.com/git/u-boot.git;branch=omap3;protocol=git \
               file://fw_env.config \
               file://pin-mux.patch;apply==yes \
              "
And bitbake it:

    cd ${OVEROTOP}
    bitbake -c clean u-boot-omap3; bitbake u-boot-omap3

And copy it to your gumstix:

**WARNING**: Be wise. Test your u-boot on an SDHC before corrupting your NAND.

    GUMSTIX=192.168.1.20 # change this to your gumstix's address
    scp ${OVEROTOP}/tmp/deploy/glibc/images/overo/u-boot-overo.bin ${GUMSTIX}:/media/mmcblk0p1/u-boot.bin


node-devreg
===============

with node-devreg you can change the registry settings in user-space

`./doc/omap3530/control.js`

    // control registers
    var padconf = {
      base_address: "0x48002030",
      description: "Pad Multiplexing Register Fields - Core Control Module Pad Configuration Register Fields - pin muxing, etc",
      pages: [764],
      registers: {
        cam_hs: {
          description: "Configuring hsync/vsync or gpio94/gpio95",
          address: "0x4800210C",
          offset: "0x000000DC",
          fields: {
            cam_hs: [15:0],
            cam_vs: [31:16]
          }
        }
      }
    }
    exports.padconf = padconf;

`control_settings.js`

    var padconf = {
      "cam_hs": {
        "cam_hs": "0",
        "cam_vs": "0"
      }
    }

Resources
=========

  * [How to build U-Boot patches for use with OE](http://www.jumpnowtek.com/index.php?option=com_content&view=article&id=59&Itemid=66)
  * [Gumstix U-Boot development without bitbake](http://www.jumpnowtek.com/index.php?option=com_content&view=article&id=55&Itemid=61)
  * [OMAP 3530 Technical Reference Manual spruf98h](http://www.ti.com/lit/pdf/spruf98)
    * p762 - Mode Selection - `0b000` Mode0, [...], `0b111` Mode7
    * p767 - `CONTROL_PADCONF_CAM_HS[15:0]` - physical address `0x4800 210C`, mode0 `cam_hs`, mode4 `gpio94`, mode5 `hw_dbg0`, mode7 `safe_mode`
    * p767 - `CONTROL_PADCONF_CAM_HS[31:16]` - physical address `0x4800 210C`, mode0 `cam_vs`, mode4 `gpio95`, mode5 `hw_dbg1`, mode7 `safe_mode`
    * p846 - `CONTROL_PADCONF_CAM_HS`, RW, 32-bit, offset `0x0000 00DC`, physical address `0x4800 210C`
    * p858
      *     REGISTER NAME, Pad Name, Physical Address, WakeUpx, OffMode, Input Enable,  Reserved, PU/PD, MuxMode
      *     `CONTROL_PADCONF_CAM_HS[15:0]`, `cam_hs`, `0x4800 210C`, `0b00`, `0b00000`, `0b1`, `0b000`, `0b01`, `0b111`
      *     `CONTROL_PADCONF_CAM_HS[31:16]`, `cam_vs`, `0x4800 210C`, `0b00`, `0b00000`, `0b1`, `0b000`, `0b01`, `0b111`
    * p1301 - Figure 12-1. Camera ISP Highlight
    * p1305 - 12.2.2 Camera ISP Signal Descriptions - (explains that this is only available in sync mode)
    * p1307 - Figure 12-2. Parallel Interface in Generic Configuration
    * p1308 - Figure 12-3. Parallel Interface in ITU-R BT.656 Configuration
    * p1309 - Figure 12-4. CSI2, CSI1 Serial Interface Configuration
    * pp1310-1312 - 12.2.4.1 Parallel Generic Configuration Protocol and Data Format (8, 10, 11, 12 Bits)
    * p1363 - Figure 12-53. Camera ISP Block Diagram
    * p1385 - SYNC mode: In this mode, the cam_hs and cam_vs signals use dedicated wires. Synchronization signals are provided by either the sensor or the camera ISP. This mode works with 8-, 10-, 11-, and 12-bit data. It supports both progressive and interlaced image-sensor modules.
    * p1459 - 12.5.5.6.1.2 Timing Generator and Frame Settings The polarities of the `cam_hs`, `cam_vs`, and `cam_fld` signals are controlled by the `CCDC_SYN_MODE[3] HDPOL`, `CCDC_SYN_MODE[2] VDPOL`, and `CCDC_SYN_MODE[4] FLDPOL` bit fields. The polarities can be positive or negative. [...] Furthermore, the directions of the `cam_fld` and `cam_hs`/`cam_vs` signals are controlled by the `CCDC_SYN_MODE[1] FLDOUT` and `CCDC_SYN_MODE[0] VDHDOUT` bits. If `CCDC_SYN_MODE[0] VDHDOUT` is set as an output, the `CCDC_PIX_LINES` register controls the length of the `cam_hs` and `cam_vs` signals.
    * p1548 - Table 12-193. `CCDC_SYN_MODE`. 
      * EXWEN The `cam_wen` signal can be used as an external memory write-enable signal. The data is stored to memory only if `cam_hs`, `cam_vs` and `cam_wen` signals are asserted.
      * HDPOL Sets the cam_hs signal polarity.
      * VDPOL cam_vs signal polarity
      * VDHDOUT cam_hs and cam_vs signal directions.
