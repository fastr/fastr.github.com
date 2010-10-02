---
layout: article
uuid: b3796d5a-d8a1-4844-a832-ba7637839892
title: Creating a custom board_overo.c
categories: gumstix embedded
updated_at: 2010-07-28
---

Goal
====

Basically I've got a custom board (based on the tobi) for the overo and I want to create a custom build for it with a custom kernel.

    bitbake custom-board-image


1000-foot overview
==================

  * Choose a kernel that compiles
  * Use it as a base for your custom kernel
  * Copy the source with all Overo patches built-in
  * Make the changes
  * Test
  * Create the necessary bitbake patches and files


Copying the bitbake skeleton for the kernel
===============

First test that the kernel you wish to use as your base bitbakes successfully

    bitbake linux-omap-psp

Then copy that base

    cd ~/overo-oe
    mkdir -p user.collection/recipes/linux/
    mkdir -p user.collection/recipes/linux/files/configs
    touch user.collection/recipes/linux/files/configs/.empty
    cp org.openembedded.dev/recipes/linux/linux.inc user.collection/recipes/linux/
    cp org.openembedded.dev/recipes/linux/multi-kernel.inc user.collection/recipes/linux/
    cp org.openembedded.dev/recipes/linux/linux-omap-psp_2.6.32 \
      user.collection/recipes/linux/linux-my-custom_2.6.32.bb
    cp -a org.openembedded.dev/recipes/linux/linux-omap-psp-2.6.32 \
      user.collection/recipes/linux/linux-my-custom-2.6.32

That is everything necessary to start a custom kernel. Test that it will actually build

    bitbake linux-my-custom


Using bitbake to get the source + patches
=======================

    cd ~/overo-oe
    bitbake linux-my-custom
    cp -a ./tmp/work/overo-angstrom-linux-gnueabi/linux-my-custom-2.6.32-r81 ./linux-my-custom
    sudo cp -a ./tmp/work/overo-angstrom-linux-gnueabi/linux-my-custom-2.6.32-r81/git/.pc ./linux-my-custom/
    cd ./linux-my-custom/
    git init
    git add .
    git commit -m "freshly patched kernel source from mainstream"

In the appendix I list a way which may work for manually getting the source + patches.


Making customizations
====================

Let's say a handy-dandy friend has made some modifications to board-overo.c and created board-overo-camera.c.

I can grab his files from my e-mail and plop them into the mix.

I'll check to see that the changes are good and I understand them

    diff -y --suppress-common-lines ~/Downloads/board-overo.c arch/arm/mach-omap2/board-overo.c

    static struct platform_device my_custom_cam_device = { <
      .name           = "my_custom_cam",                   <
      .id             = -1,                                <
    };                                                     <


Then I copy it over

    cp ~/Downloads/board-overo.c arch/arm/mach-omap2/board-overo.c
    cp ~/Downloads/board-custom-camera.c arch/arm/mach-omap2/board-custom-camera.c
    cp ~/Downloads/board-custom-camera.h arch/arm/mach-omap2/board-custom-camera.h
    cp ~/Downloads/custom-cam.c drivers/media/video/custom-cam.c
    cp ~/Downloads/config arch/arm/configs/overo_defconfig
    cp ~/Downloads/Makefile arch/arm/mach-omap2/Makefile

I can also check the differences now

    git diff

And then I commit my changes with a comment

    git status
    git add ./arch ./drivers
    git commit -m "added custom_cam support for custom_board"


Testing
=======

    PATH=~/overo-oe/tmp/sysroots/i686-linux/usr/bin/:$PATH
    #cp ~/overo-oe/tmp/sysroots/i686-linux/usr/bin/mkimage /usr/bin/
    #sudo apt-get install uboot-mkimage
    make ARCH=arm CROSS_COMPILE=~/overo-oe/tmp/cross/armv7a/bin/arm-angstrom-linux-gnueabi- overo_defconfig
    make ARCH=arm CROSS_COMPILE=~/overo-oe/tmp/cross/armv7a/bin/arm-angstrom-linux-gnueabi- menuconfig
    make ARCH=arm CROSS_COMPILE=~/overo-oe/tmp/cross/armv7a/bin/arm-angstrom-linux-gnueabi- uImage modules
    scp arch/arm/boot/uImage gumstix:/media/mmcblk0p1/

TODO: use my_custom_defconfig


Creating custom patches
=======================

Creating patches for the past two commits

    git format-patch -2

TODO: to be continued


Resources
=========

  * [board-overo.c Technote 021](http://old.nabble.com/board-overo.c-tp28833295p28838644.html)
  * [Gumstix kernel development without bitbake](http://www.jumpnowtek.com/index.php?option=com_content&view=article&id=46&Itemid=54)


Appendix
========

Manually getting the source + patches
-------------------------

In `linux-my-custom_2.6.32.bb` I found the original kernel source and I'm going to clone that and add the patches I need.

    cd ~/overo-oe
    git clone git://arago-project.org/git/people/sriram/ti-psp-omap.git
    mv ti-psp-omap linux-my-custom
    cd linux-my-custom
    mkdir patches
    cp user.collection/recipes/linux/linux-my-custom-2.6.32/*.patch patches/
    # You may also want additional board patches
    cp -a user.collection/recipes/linux/linux-my-custom-2.6.32/overo/ patches/

Now we want to apply all of those patches - here's a miniscript for it

    ls patches/*.patch | sort | while read PATCH
    do
      echo $PATCH
      git apply --stat --apply --whitespace=fix $PATCH && rm $PATCH || break
    done

In my case, a few patches didn't apply so I had to apply them by hand. No big deal - probably just a problem with whitespace and line numbers.

After all that, time to commit

    rm -rf patches
    git add .
    git commit -m "brought kernel up to mainline"


Creating a raw capture driver
-------------------

These are the files to start with

    ./drivers/media/video/Makefile
    ./drivers/media/video/Kconfig
    ./drivers/media/video/mt9t111_reg.h
    ./drivers/media/video/mt9t111.c
    
    ./include/media/mt9t111.h
    
    ./arch/arm/mach-omap2/board-omap3beagle-camera.c
    ./arch/arm/mach-omap2/board-overo-camera.c
    ./arch/arm/mach-omap2/board-overo.c
    ./arch/arm/mach-omap2/Makefile
    ./arch/arm/mach-omap2/board-omap3beagle.c
    #./arch/arm/mach-omap2/board-overo-camera.h
    
    ./arch/arm/configs/overo_defconfig
    ./arch/arm/configs/omap3_beagle_cam_defconfig
    ./arch/arm/configs/omap3_beagle_defconfig

