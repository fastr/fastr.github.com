---
layout: article
uuid: 95c7730b-8e43-4116-bf2f-ed70123173de
title: Partition MicroSDHC for Gumstix Overo
categories: u-boot gumstix
updated_at: 2010-08-19
---

Goal
====

UPDATE: fixed a bug with the number of cylinders. Now cards larger than 2gb will register their full size.

A script that can be run by a someone not familiar with the instrinsics of the Gumstix Overo to create a boot-image-ready microSDHC card of any size.

In the case that the script runs on the overo:

    sudo partition-overo-sdhc.sh /dev/mmcblk0

In the case that the script runs on another Linux box:

    sudo partition-overo-sdhc.sh /dev/sdf

Script
======

The following script automates the process described by [Gumstix Overo: Creating a bootable microSD card](http://www.gumstix.net/Setup-and-Programming/view/Overo-Setup-and-Programming/Creating-a-bootable-microSD-card/111.html)

It may be run on the Gumstix Overo or on a system with an appropriate microSDHC reader.

`/usr/local/bin/partition-overo-sdhc.sh`:

    #!/bin/bash
    #http://www.gumstix.net/Setup-and-Programming/view/Overo-Setup-and-Programming/Creating-a-bootable-microSD-card/111.html

    # set this script to exit at any point an operation returns failure
    set -e

    # Create the environment for this script
    DEV=$1
    DEFAULT_DEV=/dev/mmcblk0
    IS_OVERO=`cat /proc/cpuinfo | grep Overo || true`
    IS_MMCBLK=`echo ${DEV} | grep /dev/mmcblk || true`

    if [ ! -n "${DEV}" ]
    then
      if [ -n "${IS_OVERO}" ]
      then
        echo "About to partition (DESTROY data) ${DEFAULT_DEV}"
        DEV=${DEFAULT_DEV}
        echo "CTRL+C now if that's not what you wanted..."
        sleep 3
        echo "continuing..."
      else
        echo -n 'Usage: `'$0' /dev/MICRO_SDHC_BLOCK_DEVICE`'
        exit 1;
      fi
    fi

    # Attempt to ensure that the disk is not in use
    umount ${DEV} 2>/dev/null || true
    umount ${DEV} 2>/dev/null || true

    # Backup the MBR - in case the working hard drive is specified by mistake, rather than the sd card
    dd if=$DEV of=./mbr.bak bs=512 count=1 2> /dev/null
    echo "Created backup of MBR... just to be safe"
    echo 'use `dd of='${DEV}' if=./mbr.bak bs=512 count=1` to restore or `rm mbr.bak` to remove'

    # Calculate what the number of cylinders should be according to the overo uboot specs
    let BYTES=`fdisk $DEV -l | grep Disk | grep bytes | cut -d',' -f2 | cut -d' ' -f2`
    CYL=$(($BYTES/255/63/512))

    # This scripts fdisk to run in expert mode
    # it changes the head, sector, and cylinder, to match those required for the overo uboot
    # it then creates a 32mb FAT boot partition and uses the rest of the card for the rootfs
    #
    # Comments may not be placed between the EOF blocks below
    fdisk $DEV << EOF >/dev/null 2>/dev/null || true
    o
    x
    h
    255
    s
    63
    c
    $CYL
    r
    n
    p
    1
    1
    +32M
    t
    c
    a
    1
    n
    p
    2
    6
    
    w
    EOF
    # The return of this process is almost always false due to that the kernel cannot resync the
    # partition table.
    # A reboot (on the overo) or unplugging and replugging the sdhc (on a Linux box)
    # is required before continuing on to copy over the SD image.

    # Overo Note:
    # if the `reboot` command doesn't work use
    # echo b > /proc/sysrq-trigger
    # which is similar to a hard power cycle

    echo -n "Probably prepared SD card successfully..."
    if [ -n "${IS_OVERO}" ] && [ -n "${IS_MMCBLK}"  ]
    then
      echo "The overo should be rebooted (to resync the partition table with the kernel) because you can't hot-swap/reinsert the microSDHC."
      echo 'Try `reboot` (soft, but sometimes hangs) or `cd /; umount -a; sync; echo b > /proc/sysrq-trigger` (power cycle)'
    else
      echo "The microSDHC must be removed from and reinserted into the card reader to continue"
    fi
