---
layout: article
title: Export gpio pins on gumstix
categories: gpio gumstix
updated_at: 2010-08-03
---

Goal
====

Be able to twiddle a gpio line by hand.

Resources
========

SPRS507F (OMAP3530/25 Applications Processor)

  * Table 2-1. Ball Characteristics (CBB Pkg.) - Page 28-69 -  GPIO pins, mux modes, etc

Caution
=======

If the gpio is already in use by another device in a different mode the results are undefind - and in some cases could cause physical damage to the board (creating a short, for example).

In the kernel via board-overo.c
===============================

You might take a glance at [Table 2-2. Ball Characteristics (CBC Pkg.)](http://focus.ti.com/lit/ds/symlink/omap3530.pdf?DCMP=dsps_omap3530prf__090819&HQS=Other+OT+OMAP3530perf-increase-prdatash) to see which gpio pins are available for use.

If the GPIO you would like to export is pin-muxed, you must enable `CONFIG_OMAP_MUX` by running `menuconfig` or editing `overo_defconfig`

Create a function to export the gpios you need in `KERNDIR/arch/arm/mach-omap2/board-overo.c` check my other post about custom board files to learn where `KERNDIR` might be.

    #define OVERO_GPIO_CAM_HS 94
    
    static void __init test_custom_gpio_init(void)
    {
      printk("... initing test_custom_gpio\n");
      if ((gpio_request(OVERO_GPIO_CAM_HS, "OVERO_GPIO_CAM_HS") == 0) &&
          (gpio_direction_output(OVERO_GPIO_CAM_HS, 1) == 0)) {
        if (0 == gpio_export(OVERO_GPIO_CAM_HS, 0)) {
          printk("exported test_custom_gpio 94");
        } else {
          printk("didn't export test_custom 94");
        }
      } else {
        printk(KERN_ERR "could not obtain test_custom gpio for "
                            "OVERO_GPIO_CAM_HS\n");
      }
    }

Then add that function to the list in `overo_init`

    static void __init overo_init(void)
    {
      omap3_mux_init(board_mux, OMAP_PACKAGE_CBB);
      overo_i2c_init();
      platform_add_devices(overo_devices, ARRAY_SIZE(overo_devices));
      omap_serial_init();
      overo_flash_init();
      usb_musb_init();
      usb_ehci_init(&ehci_pdata);
      overo_spi_init();
      overo_init_smsc911x();
      overo_display_init();
      test_custom_gpio_init(); // our gpio function

    // ... lots more stuff
    }

In user-space via `/sys/class/gpio/export`
=====================

Note that this cannot be mixed with the in-kernel method method

    echo 94 > /sys/class/gpio/export
    echo 94 > /sys/class/gpio/unexport

In user-space via `/dev/mem`
===========

TODO

Via uboot
=========

TODO
