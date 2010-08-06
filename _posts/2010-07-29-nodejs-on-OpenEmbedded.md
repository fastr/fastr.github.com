---
layout: article
title: nodejs on OpenEmbedded
categories: uncategorized
updated_at: 2010-07-29
---

Goal
====

I would like to be able to cross-compile nodejs using bitbake. If you're interested in building it natively on a Gumstix Overo (shared or static), that's done. Google my other post of the same title.

    bitbake nodejs
    opkg install nodejs

Current Status
==============

The big problem is that I've got to figure out how to compile v8 which uses scons, which only about 4 other packages use.

Files
=====

These are the files I have so far

nodejs_0.1.102.bb
-----------------

    DESCRIPTION = "nodeJS Evented I/O for V8 JavaScript"
    PR = "r0"
    DEPENDS = "openssl"
    SRC_URI = " \
    http://nodejs.org/dist/node-v${PV}.tar.gz \
    "
    SRC_URI[md5sum] = "93279f1e4595558dacb45a78259b7739"
    SRC_URI[sha256sum] = "bd9b1d09ad40ceaef4bdd46019960c5c2fe87026c9598a6fb23c66457510a22d"
    S = "${WORKDIR}/node-v${PV}"
    do_install () {
    install -d ${D}${bindir}/
    install -m 0755 ${S}/node ${D}${bindir}/
    }
    FILES_${PN} = "${bindir}/node"


nodejs-static_0.1.102.bb
-----------------

    DESCRIPTION = "nodeJS Evented I/O for V8 JavaScript"
    PR = "r0"
    DEPENDS = "openssl-static"
    SRC_URI = " \
    http://nodejs.org/dist/node-v${PV}.tar.gz \
    "
    SRC_URI[md5sum] = "93279f1e4595558dacb45a78259b7739"
    SRC_URI[sha256sum] = "bd9b1d09ad40ceaef4bdd46019960c5c2fe87026c9598a6fb23c66457510a22d"
    S = "${WORKDIR}/node-v${PV}"
    do_install () {
    install -d ${D}${bindir}/
    install -m 0755 ${S}/node ${D}${bindir}/
    }
    FILES_${PN} = "${bindir}/node"


Patches
=======

./node-v0.1.102/deps/libev/wscript
---------------------

    - code = """
    -     #include <syscall.h>
    -     #include <time.h>
    -     #include <stdio.h>
    -
    -     int main() {
    -         struct timespec ts;
    -         int status = syscall(SYS_clock_gettime, CLOCK_REALTIME, &ts);
    -         return 0;
    -     }
    - """
    - conf.check_cc(fragment=code, define_name="HAVE_CLOCK_SYSCALL", execute=True,
    -               msg="Checking for SYS_clock_gettime")
    + conf.define('HAVE_CLOCK_SYSCALL', 1)

