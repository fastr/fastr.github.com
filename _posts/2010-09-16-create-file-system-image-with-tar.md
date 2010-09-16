---
layout: article
title: create file system image with tar
categories: utility
updated_at: 2010-09-16
---
Goal
====

A script for backing up an sdhc rootfs as a tarball.

Script
====

`create-rootfs-image-from-sdhc.sh`:

    #!/bin/bash

    set -e

    SDHC=$1
    MOUNT=/mnt/new_rootfs
    PACKAGE=/home/shared/sdhc-rootfs-`date +%F`.tar
    PACKAGE_LINK=/home/shared/sdhc-rootfs.tar

    if [ ! -n "${SDHC}" ] || [ ! "0" == "`id -u`" ]
    then
      echo 'Usage: `sudo '$0' /dev/MICRO_SDHC_PARTITION`'
      echo 'Example: `sudo '$0' /dev/sdc1`'
      echo 'most likely one of the below:'
      ls /dev/sd[b-z]* | grep [0-9]
      exit 1;
    fi

    # Create the environment for this script

    umount ${SDHC} ${MOUNT} 2> /dev/null || true
    mkdir -p ${MOUNT}
    mount ${SDHC} ${MOUNT}
    cd ${MOUNT}

    echo "Creating backup of ${SDHC}"
    tar cf ${PACKAGE} ./
    ln -sf ${PACKAGE} ${PACKAGE_LINK}

    sync
    cd /tmp
    umount ${MOUNT}
    rm ${MOUNT} -rf
