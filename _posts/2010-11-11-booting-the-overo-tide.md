---
layout: article
uuid: ac7784a4-cd1d-4850-8b64-f3c2b150e153
title: booting the Overo Tide
name: booting-the-overo-tide
created_at: 2010-11-11
updated_at: 2010-11-11
categories: u-boot
---
u-boot.bin and MLO
====

The Overo Tide is brand new and you can't boot it with an older `x-loader`.
You must have a recent version of `u-boot.bin` and `MLO`.

    cd ~/overo-oe
    bitbake -c clean u-boot-omap3 x-load
    bitbake u-boot-omap3 x-load
    ls tmp/deploy/glibc/images/overo/
    # note MLO-overo and u-boot-overo.bin

saveenv
====

The Tide has no NAND so `saveenv` just won't work.
If you feel the need to roll your own u-boot, be my guest.

However, you can also create a `myubootenv.cmd` script and turn it into a `boot.scr`.

`myubootenv.cmd`:
    setenv ethaddr 00:00:00:FF:FF:FF
    setenv mpurate 720
    setenv vram 4M
    setenv linuxmem 176M
    setenv mmcargs 'setenv bootargs console=${console} mpurate=${mpurate} vram=${vram} mem=${linuxmem} omapfb.mode=dvi:${dvimode} omapfb.debug=y omapdss.def_disp=${defaultdisplay} root=${mmcroot} rootfstype=${mmcrootfstype}'
    setenv bootcmd 'mmc init; run loaduimage; run mmcboot'
    boot

And add the binary headers:

    mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "myubootenv" -d myubootenv.cmd boot.scr

common u-boot / kernel parameters
====

  * `bootcmd` is what `u-boot` will execute when the `boot` command is issued
  * `mpurate` directly sets the ARM clock rate and also indirectly sets the DSP clock rate.
  * `vram` is the amount of shared video memory to be taken from the system memory
  * `mem` limits the amount of memory - useful if you are using the DSP - `cmemk`, `dsplink`, etc
  * `ethaddr` overrides the network card's firmware MAC address, primarily for PXE / tftp / network booting.
    * useful when your vendor doesn't set a MAC for you and you don't care to flash your NICs firmware.
    * using `ifconfig eth0 hw ether` in Linux will override this setting.
    * **WARNING** duplicate MAC addresses on your network will lead to frustrating and seemingly mysterious network bugs
