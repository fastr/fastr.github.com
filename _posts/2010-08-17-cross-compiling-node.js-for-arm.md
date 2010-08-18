---
layout: article
title: cross compiling node.js for arm
categories: nodejs
updated_at: 2010-08-17
---
Goal
====

Quickly build nodejs for an ARM target system using a pre-built toolchain. The first part of these instructions should apply to most cross-compiling situations.

Cross-compiling
=========

You'll need a prebuilt toolchain. I'm going to suggest CodeSourcery because TI uses them. The caveat is that their versions of `libc` and such may not fit well with your linux-arm distribution.

CodeSourcery `arm-none-linux-gnueabi-` Toolchain
---------

  * [Direct download](http://www.codesourcery.com/sgpp/lite/arm/portal/package4571/public/arm-none-linux-gnueabi/arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2)

    mkdir /opt/code-sourcery/
    tar xf arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2 -C /opt/code-sourcery/

Set Environment
----------

Copied from OpenEmbedded and modified for CodeSourcery. This can probably still be pruned a little bit, but it's good enough for now for me.

Sometimes this `exit`s right away. I don't know why. Running it again seems to solve the problem.

`cross-compiler-shell.sh`

    #!/bin/sh -e
    export CSTOOLS=/opt/code-sourcery/arm-2009q1
    export CSTOOLS_INC=${CSTOOLS}/arm-none-linux-gnueabi/libc/usr/include
    export CSTOOLS_LIB=${CSTOOLS}/arm-none-linux-gnueabi/libc/usr/lib
    export TARGET_ARCH="-march=armv7-a" # must be at least armv5te
    export TARGET_TUNE="-mtune=cortex-a8 -mfpu=neon -mfloat-abi=softfp -mthumb-interwork -mno-thumb" # optional
    
    export CPP="arm-none-linux-gnueabi-gcc -E"
    export STRIP="arm-none-linux-gnueabi-strip"
    export OBJCOPY="arm-none-linux-gnueabi-objcopy"
    export AR="arm-none-linux-gnueabi-ar"
    export F77="arm-none-linux-gnueabi-g77 ${TARGET_ARCH} ${TARGET_TUNE}"
    unset LIBC
    export RANLIB="arm-none-linux-gnueabi-ranlib"
    export LD="arm-none-linux-gnueabi-ld"
    export LDFLAGS="-L${CSTOOLS_LIB} -Wl,-rpath-link,${CSTOOLS_LIB} -Wl,-O1 -Wl,--hash-style=gnu"
    export MAKE="make"
    export CXXFLAGS="-isystem${CSTOOLS_INC} -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3 -fpermissive -fvisibility-inlines-hidden"
    export LANG="en_US.UTF-8"
    export HOME="/home/peru"
    export CCLD="arm-none-linux-gnueabi-gcc ${TARGET_ARCH} ${TARGET_TUNE}"
    export PATH="${CSTOOLS}/bin:/opt/code-sourcery/arm-2009q1/bin/:${HOME}/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games"
    export CFLAGS="-isystem${CSTOOLS_INC} -fexpensive-optimizations -frename-registers -fomit-frame-pointer -O2 -ggdb3"
    export OBJDUMP="arm-none-linux-gnueabi-objdump"
    export CPPFLAGS="-isystem${CSTOOLS_INC}"
    export CC="arm-none-linux-gnueabi-gcc ${TARGET_ARCH} ${TARGET_TUNE}"
    export TITOOLSDIR="/mnt/data/overo-oe/ti"
    export TERM="screen"
    export SHELL="/bin/bash"
    export CXX="arm-none-linux-gnueabi-g++ ${TARGET_ARCH} ${TARGET_TUNE}"
    export NM="arm-none-linux-gnueabi-nm"
    export AS="arm-none-linux-gnueabi-as"

    bash --norc

    
Patch node.js for cross-compilation
---------

    mkdir ~/src
    cd ~/src
    wget http://nodejs.org/dist/node-v0.1.104.tar.gz
    tar xf node*
    cd node*
    cp deps/libev/wscript deps/libev/wscript.orig
    cp wscript wscript.orig

Patch `libev/wscript`

    --- a/deps/libev/wscript
    +++ b/deps/libev/wscript
    @@ -41,6 +41,7 @@ def configure(conf):
         conf.check_cc(header_name="sys/eventfd.h", function_name="eventfd")


    +  ''' Can't run cross-binary code
       code = """
           #include <syscall.h>
           #include <time.h>
    @@ -54,6 +55,8 @@ def configure(conf):
       """
       conf.check_cc(fragment=code, define_name="HAVE_CLOCK_SYSCALL", execute=True,
                     msg="Checking for SYS_clock_gettime")
    +  '''
    +  conf.define('HAVE_CLOCK_SYSCALL', 1)

       have_librt = conf.check(lib='rt', uselib_store='RT')
       if have_librt:


    vim wscript

    --- a/wscript
    +++ b/wscript
    @@ -319,11 +319,15 @@ def v8_cmd(bld, variant):
       if bld.env['DEST_CPU'] == 'x86_64':
         arch = "arch=x64"

    + cross_arch = False
    + # TODO would use -1 != str.find('linux-gnueabi'), but this is sometimes a string and other times an array
    + # if bld.env['AR'] == 'arm-angstrom-linux-gnueabi-ar':
    + #   arch = "arch=arm"
    + #   cross_arch = True
    + # 
    + arch = "arch=arm"
    + cross_arch = True

       if variant == "default":
         mode = "release"
       else:
         mode = "debug"

    +  snapshot = 'snapshot=on'
    +  if cross_arch:
    +    snapshot = ''
    -  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static snapshot=on'
    +  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static ' + snapshot

Cross-compile nodejs
----------

    cd ~/src/node*
    ~/cross-compiler-shell.sh
    # prompt changes to a boring one without color
    ./configure --without-ssl # the codesourcery toolchain doesn't include openssl, but node mistakenly detected it
    # Note you should see `arm-none-linux-gnueabi-gcc` instead of `gcc`
    make
