---
layout: article
uuid: 4c1d520a-24fd-4f31-9eca-5b0b2154fe5e
title: gumstix with upstart
categories: scratchpad gumstix
updated_at: 2010-09-18
---
Goal
====

Use `upstart` rather than the default `sysvinit`.

Status: incomplete

Procedure
====

    cd ~/overo-oe
    bitbake omap3-console-image
    bitbake time upstart
      # time - tmp/deploy/glibc/ipk/armv7a/time_1.7-r1.6_armv7a.ipk
      # libupstart0 - tmp/deploy/glibc/ipk/armv7a/libupstart0_0.3.11-r1.6_armv7a.ipk
      # upstart-sysvcompat - scp tmp/deploy/glibc/ipk/armv7a/upstart-sysvcompat_0.3.11-r1.6_armv7a.ipk
      # upstart - tmp/deploy/glibc/ipk/armv7a/upstart_0.3.11-r1.6_armv7a.ipk

Error (hack) fixing:

    rm /usr/sbin/readprofile
