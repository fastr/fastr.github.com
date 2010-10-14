---
layout: article
uuid: 66af2e1d-7f9c-43b9-950b-0a1eddde0f87
title: configure bitbake kernel (remotely)
categories: gumstix bitbake
updated_at: 2010-09-18
---
Goal
====

Custom configure a kernel for gumstix overo.

In particular, I wanted to add support for `ext4` and `gpt` (as opposed to `mbr`)

Procedure
====

Begin to configure the kernel

    cd ~/overo-oe
    REV=34 # CHANGE THIS to be the current kernel revision
    bitbake linux-omap3-2.6.${REV} -c clean
    bitbake linux-omap3-2.6.${REV} -c configure

Get into the bitbake environment.

    cd tmp/work/overo-angstrom-linux-gnueabi/linux-omap3-2.6.${REV}-r*/git
    cp ../temp/run.do_configure.* ./
    vim run.do_configure.*
    # replace the last line `do_configure' with `bash --norc`
    # :%s/^do_configure$/bash --norc/g
    ./run.do_configure.*

Now at the `bash-4.0$` prompt (which has all of the cross-compile variables pre-set)

    make menuconfig

Once you've made the changes you need you may want to save the `.config`

Now run bitbake again

    bitbake bitbake linux-omap3-2.6.${REV}

or

    bitbake linux-omap3-2.6.34 -f -c compile
    bitbake linux-omap3-2.6.34 -f -c deploy

And install onto the gumstix's microSDHC:

    IP=192.168.1.20 # CHANGE THIS to the gumstix' ip address
    scp tmp/deploy/glibc/images/overo/uImage-overo.bin root@${IP}:/media/mmcblk0p1/uImage

Note: `/media/mmcblk0p1` will mount by default with the `sync` option, which makes transfer terribly slow.

    
Appendix
====

For me, `bitbake linux-omap3-2.6.34 -c menuconfig` results in:

    NOTE: Running task 368 of 368 (ID: 5, /home/harry/overo-oe/org.openembedded.dev/recipes/linux/linux-omap3_2.6.34.bb, do_menuconfig)
    ERROR: function do_menuconfig failed
    ERROR: log data follows (/home/harry/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/linux-omap3-2.6.34-r89/temp/log.do_menuconfig.16568)
    | Xlib:  extension "Generic Event Extension" missing on display "localhost:11.0".
    | Xlib:  extension "RANDR" missing on display "localhost:11.0".
    | Xlib:  extension "Generic Event Extension" missing on display "localhost:11.0".
    | )
    | GConf Error: Failed to contact configuration server; some possible causes are that you need to enable TCP/IP networking for ORBit, or you have stale NFS locks due to a system crash. See http://projects.gnome.org/gconf/ for information. (Details -  1: Failed to get connection to session: /bin/dbus-launch terminated abnormally with the following error: Autolaunch requested, but X11 support not compiled in.
    | Cannot continue.
    | )
    | **
    | ERROR:terminal-app.c:1444:terminal_app_init: assertion failed: (app->default_profile_id != NULL)
    | /home/harry/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/linux-omap3-2.6.34-r89/temp/run.do_menuconfig.16568: line 1223: 16570 Aborted                 gnome-terminal --disable-factory -t "$TERMWINDOWTITLE" -x $SHELLCMDS
    NOTE: Task failed: /home/harry/overo-oe/tmp/work/overo-angstrom-linux-gnueabi/linux-omap3-2.6.34-r89/temp/log.do_menuconfig.16568
