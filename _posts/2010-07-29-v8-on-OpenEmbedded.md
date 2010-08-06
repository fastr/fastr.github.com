---
layout: article
title: v8 on OpenEmbedded
categories: uncategorized
updated_at: 2010-07-29
---

Goal
====

I would like to be able to cross-compile v8 using bitbake.

If you're interested in building it natively on a Gumstix Overo, that's done. Google my first post about node.JS on OpenEmbedded.

    bitbake v8
    opkg install v8

Current Status
==============

There appears to be an issue with scons.bbclass or the v8 SConfig. I've e-mailed the [OE] mailing list and am awaiting reply.

v8_r5144.bb
-----------

    DESCRIPTION = "V8 JavaScript Engine"
    PR = "r0"
    DEPENDS = ""
    SRC_URI = " \
    svn://v8.googlecode.com/svn;module=trunk;proto=http;rev=5144 \
    "
    S = "${WORKDIR}/trunk"
    FILES_${PN} = "${bindir}/libv8"

    inherit scons

Resources
=========

  * http://kynesim.blogspot.com/2009/10/muddle-cross-compiling-v8.html
