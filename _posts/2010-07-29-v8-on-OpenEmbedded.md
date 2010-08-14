---
layout: article
title: v8 on OpenEmbedded
categories: v8 scons
updated_at: 2010-08-14
---

Goal
====

I would like to be able to cross-compile v8 using bitbake.

If you're interested in building it natively on a Gumstix Overo, that's done. Google my first post about node.JS on OpenEmbedded.

    bitbake libv8
    opkg install libv8

Current Status
==============

scons doesn't use environment variables, so trying to get it working "the scons way" TM seems a lost cause, 
but I'll check on the scons, v8, and bitbake mailing list with my proposed method to be sure.

However, with a little Makefile magic I can get it to compile well enough it seems. 

`${OVEROTOP}/user.collection/recipes/libv8/libv8_r5266.bb`
-----------

    DESCRIPTION = "V8 JavaScript Engine"
    PR = "r0"
    DEPENDS = ""
    SRC_URI = " \
    svn://v8.googlecode.com/svn;module=trunk;proto=http;rev=5266 \
    file://overo-opts.patch \
    file://Makefile \
    "
    S = "${WORKDIR}/trunk"
    FILES_${PN} = "${bindir}/libv8"

    #inherit scons # doesn't currently provide meaningful setup

`${OVEROTOP}/user.collection/recipes/libv8/files/Makefile`
------------

    all:
      CC=`which ${CC}` CXX=`which ${CXX}` AR=`which ${AR}` RANLIB=`which ${RANLIB}` scons arch=arm
    
    .PHONY: all


`${OVEROTOP}/user.collection/recipes/libv8/files/overo-opts.patch`
------------

    diff --git trunk/SConstruct.orig trunk/SConstruct
    index 0abaeed..b85262a 100644
    --- trunk/SConstruct.orig
    +++ trunk/SConstruct
    @@ -42,6 +42,13 @@ ANDROID_TOP = os.environ.get('TOP')
     if ANDROID_TOP is None:
       ANDROID_TOP=""

    +# OVERO_TOP is the top of the Overo checkout, fetched from the environment
    +# variable 'OVERO_TOP'.   You will also need to set the CXX, CC, AR and RANLIB
    +# environment variables to the cross-compiling tools.
    +OVERO_TOP = os.environ.get('OVEROTOP')
    +if OVERO_TOP is None:
    +  OVERO_TOP=""
    +
     # ARM_TARGET_LIB is the path to the dynamic library to use on the target
     # machine if cross-compiling to an arm machine. You will also need to set
     # the additional cross-compiling environment variables to the cross compiler.
    @@ -108,6 +115,9 @@ ANDROID_LINKFLAGS = ['-nostdlib',
                          ANDROID_TOP + '/prebuilt/linux-x86/toolchain/arm-eabi-4.4.0/lib/gcc/arm-eabi/4.4.0/interwork/libgcc.a',
                          ANDROID_TOP + '/out/target/product/generic/obj/lib/crtend_android.o'];

    +OVERO_FLAGS = ANDROID_FLAGS # Verizon Droids are armv7-a cortex-a8, I suppose that should be similar enough
    +OVERO_INCLUDES = "" # TODO
    +
     LIBRARY_FLAGS = {
       'all': {
         'CPPPATH': [join(root_dir, 'src')],
    @@ -197,6 +207,14 @@ LIBRARY_FLAGS = {
                            '-Wstrict-aliasing=2'],
           'CPPPATH':      ANDROID_INCLUDES,
         },
    +    'os:overo': {
    +      'CPPDEFINES':   ['V8_TARGET_ARCH_ARM', '__ARM_ARCH_5__', '__ARM_ARCH_5T__',
    +                       '__ARM_ARCH_5E__', '__ARM_ARCH_5TE__'],
    +      'CCFLAGS':      OVERO_FLAGS,
    +      'WARNINGFLAGS': ['-Wall', '-Wno-unused', '-Werror=return-type',
    +                       '-Wstrict-aliasing=2'],
    +      'CPPPATH':      OVERO_INCLUDES,
    +    },
         'arch:ia32': {
           'CPPDEFINES':   ['V8_TARGET_ARCH_IA32'],
           'CCFLAGS':      ['-m32'],
    @@ -204,6 +222,8 @@ LIBRARY_FLAGS = {
         },
         'arch:arm': {
           'CPPDEFINES':   ['V8_TARGET_ARCH_ARM'],
    +      'CCFLAGS':      OVERO_FLAGS,
    +      'CPPPATH':      OVERO_INCLUDES,
           'unalignedaccesses:on' : {
             'CPPDEFINES' : ['CAN_USE_UNALIGNED_ACCESSES=1']
           },
    @@ -673,7 +693,7 @@ SIMPLE_OPTIONS = {
         'help': 'the toolchain to use (' + TOOLCHAIN_GUESS + ')'
       },
       'os': {
    -    'values': ['freebsd', 'linux', 'macos', 'win32', 'android', 'openbsd', 'solaris'],
    +    'values': ['freebsd', 'linux', 'macos', 'win32', 'android', 'openbsd', 'overo', 'solaris'],
         'default': OS_GUESS,
         'help': 'the os to build for (' + OS_GUESS + ')'
       },

    

Manual Cross-Compile
=====

Here's a way to cross-compile without any patches or fully using bitbake:

    bitbake v8
    # fails, of course
    cd ${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/libv8-r5266-r0/trunk
    USER=my-user-name \
    CROSS_DIR=/home/${USER}/overo-oe/tmp/cross/armv7a/bin \
    CC="${CROSS_DIR}/arm-angstrom-linux-gnueabi-gcc -march=armv7-a -mtune=cortex-a8" \
    CXX="${CROSS_DIR}/arm-angstrom-linux-gnueabi-g++ -march=armv7-a -mtune=cortex-a8" \
    scons arch=arm

Or without bitbake at all (you could even use your `trusty arm-none-linux-gnueabi-xyz`):

    cd ~/
    svn checkout svn://v8.googlecode.com/svn v8-read-only
    cd v8-read-only
    USER=my-user-name \
    CROSS_DIR=/home/${USER}/overo-oe/tmp/cross/armv7a/bin \
    CROSS_COMPILE=${CROSS_DIR}/arm-angstrom-linux-gnueabi- \
    CC="${CROSS_COMPILE}gcc -march=armv7-a -mtune=cortex-a8" \
    CXX="${CROSS_COMPILE}g++ -march=armv7-a -mtune=cortex-a8" \
    scons arch=arm

And how to add an OS

    cd 
    git diff --no-prefix SConstruct.orig SConstruct > overo-opts.patch
    

TODO
====

Figure out which of these to use and which not to in `SConstruct`

I'm guessing I should set the LDFLAGS all the same for the overo libs, but leave the android optimized CFLAGS

    BUILD_CFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include -O2 -g'
    BUILD_CPPFLAGS=-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include
    BUILD_CXXFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include -O2 -g -fpermissive'
    BUILD_LDFLAGS='-L/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/lib -Wl,-rpath-link,/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/lib -Wl,-rpath,/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/lib -Wl,-O1'
    CFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3'
    CPPFLAGS=-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include
    CXXFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3 -fpermissive -fvisibility-inlines-hidden'
    EXTRA_OEMAKE=' -e MAKEFLAGS='
    LDFLAGS='-L/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/lib -Wl,-rpath-link,/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/lib -Wl,-O1 -Wl,--hash-style=gnu'
    SDK_CFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include -isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3'
    SDK_CPPFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include -isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include'
    SDK_CXXFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/include -isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3 -fpermissive'
    SDK_LDFLAGS='-L/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/lib -Wl,-rpath-link,/home/${USER}/overo-oe/tmp/sysroots/i686-linux/usr/lib -Wl,-O1'
    TARGET_CFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3'
    TARGET_CPPFLAGS=-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include
    TARGET_CXXFLAGS='-isystem/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/include -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3 -fpermissive'
    TARGET_LDFLAGS='-L/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/lib -Wl,-rpath-link,/home/${USER}/overo-oe/tmp/sysroots/armv7a-angstrom-linux-gnueabi/usr/lib -Wl,-O1 -Wl,--hash-style=gnu'
