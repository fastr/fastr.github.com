---
layout: article
title: Format MicroSDHC for Gumstix Overo
categories: gumstix u-boot
updated_at: 2010-07-30
---

Goal
====

A script to run after the microSDHC has been partitioned properly for the gumstix overo to install the over-image of choice from a host system.

In the case that the script runs on the overo:

    sudo install-overo-image-to-sdhc.sh /dev/mmcblk0

In the case that the script runs on another Linux box:

    sudo install-overo-image-to-sdhc.sh /dev/sdf


Pre-Requisists
--------------

  * The host (server) system should have already run `bitbake omap3-console-image`
  * The microSDHC should have been partitioned appropriately

Script
======

install-overo-image-to-sdhc.env
-----------

There are quite a few variables here - more than just the card to install to.

`/usr/local/bin/install-overo-image-to-sdhc.env`:

    # About the host system
    USER=coolaj86
    HOST=192.168.1.20
    PORT=22
    export SCP_HOST="-P ${PORT} ${USER}@${HOST}"
    export NET_CONF=/home/${USER}/Code/development/main/overo/etc/network/interfaces
    
    
    # Where ~/overo-oe can be found
    OE_PATH=/home/${USER}
    export OVERO_PATH=${OE_PATH}/overo-oe/tmp/deploy/glibc/images/overo
    
    
    # Boot options
    KERNEL=''
    FORMAT=tar.bz2
    export MLO=${OVERO_PATH}/MLO-overo
    export UBOOT=${OVERO_PATH}/u-boot-overo.bin
    export UIMAGE=${OVERO_PATH}/uImage-${KERNEL}overo.bin
    export ROOTFS=omap3-console-image-overo.${FORMAT}
    
    
    # Create the environment for this script
    export MNT=/mnt/overo_tmp
    
    
    # NOTE /dev/shm is way faster but may not be available once the filesystem size is large
    # TODO TMDIR should not be allowed to be MNT
    # TODO this script should not run from MNT
    export TMPDIR=/dev/shm


`/usr/local/bin/install-overo-image-to-sdhc.sh`:


    #!/bin/bash
    #http://www.gumstix.net/Setup-and-Programming/view/Overo-Setup-and-Programming/Creating-a-bootable-microSD-card/111.html

    # set this script to exit at any point an operation returns failure
    set -e

    # Create the environment for this script
    source install-overo-image-to-sdhc.env

    DEV=$1
    DEFAULT_DEV=/dev/mmcblk0
    IS_OVERO=`cat /proc/cpuinfo | grep Overo || true`
    IS_MMCBLK=`echo ${DEV} | grep /dev/mmcblk || true`

    if [ ! -n "${DEV}" ]
    then
      if [ -n "${IS_OVERO}" ]
      then
        echo "About to format (DESTROY data) ${DEFAULT_DEV}"
        DEV=${DEFAULT_DEV}
        echo "CTRL+C now if that's not what you wanted..."
        sleep 3
        echo "continuing..."
      else
        echo -n 'Usage: `'$0' /dev/MICRO_SDHC_BLOCK_DEVICE`'
        exit 1;
      fi
    fi

    # About the target system
    # Matches mmcblk0p1 disk0s1 sdf1 - OpenEmbedded, Ubuntu, BSD
    BOOT=${DEV}'*1'
    ROOT=${DEV}'*2'


    # Attempt to ensure that the disk is not in use
    umount $DEV* 2>/dev/null || true
    umount $DEV* 2>/dev/null || true

    # Ready the mount point
    umount $MNT || true
    rm -rf $MNT
    mkdir -p $MNT


    # TODO this will fail due to -P
    # TODO give overo units a sandboxed ssh key 
    # or use curl to initiate a reverse connection
    # ssh-copy-id $SCP_HOST

    # Prepare FAT boot partition
    # MLO *must* be installed first
    mkfs.vfat -F 32 $BOOT -n FAT
    mount $BOOT $MNT
    scp $SCP_HOST:$MLO $MNT/MLO
    scp $SCP_HOST:$UBOOT $MNT/u-boot.bin
    scp $SCP_HOST:$UIMAGE $MNT/uImage
    echo "sync-ing... this may take several seconds"
    umount $MNT
    umount $BOOT 2>/dev/null || true # overo auto-mounts in /media sometimes


    # Prepare rootfs
    mkfs.ext3 ${ROOT}
    mount ${ROOT} /${MNT}
    scp ${SCP_HOST}:${OVERO_PATH}/${ROOTFS} ${TMPDIR}/${ROOTFS}
    tar xvjf ${TMPDIR}/${ROOTFS} -C $MNT #BAH! Takes forever!
    rm ${TMPDIR}/${ROOTFS}


    # Configure rootfs with any special sauce we might need
    scp ${SCP_HOST}:${NET_CONF} ${MNT}/etc/network/
    ln -s "/sbin/dhclient" ${MNT}/sbin/dhclient3 2>/dev/null || true
    ln -s "/var/lib/dhcp" ${MNT}/var/lib/dhcp3 2>/dev/null || true


    # sync and umount
    echo "sync-ing... this may take several minutes"
    umount $MNT
    umount $ROOT 2>/dev/null || true


    # Cleanup any mess we madermdir $MNT || true
    #rm -rf $MNT

    # sync and reboot!
    sync
    sync
    #echo 1 > /proc/sys/kernel/sysrq

    echo "copied sd card successfully"


    # if the process fails, enter uboot and `run nandboot` to boot the nand rather than the sd card
    # Kernel from NAND, RootFS from MMC
    # setenv bootargs console=${console} mpurate=${mpurate} vram=${vram} omapfb.mode=dvi:${dvimode} omapfb.debug=y omapdss.def_disp=${defaultdisplay} root=${mmcroot} rootfstype=${mmcrootfstype}
    # setenv bootargs console=${console} mpurate=${mpurate} vram=${vram} omapfb.mode=dvi:${dvimode} omapfb.debug=y omapdss.def_disp=${defaultdisplay} root=${mmcroot} rootfstype=${mmcrootfstype}
    # nand read ${loadaddr} 280000 400000; bootm ${loadaddr}
    # the first boot takes forever to load
    # mixedboot=echo Kernel on nand, RootFS on mmc...; run mmcargs; nand read ${loadaddr} 280000 400000; bootm ${loadaddr}


