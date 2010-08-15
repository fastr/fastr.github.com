---
layout: article
title: bitbake node - Node.js on OpenEmbedded
categories: nodejs bitbake
updated_at: 2010-08-14
---
Goal
====

I would like to be able to cross-compile nodejs using bitbake. (I've already got it [natively compiled on the Gumstix Overo](/articles/bitbake-node.html))

    bitbake node
    opkg install node

Current Status
==============

It compiles!!! Now I just have to finish packaging it as a bitbake recipe.

Files
=====

These are the files I have so far. They still need a few tweaks before I create a bitbake recipe.

node_0.1.104.bb
-----------------

    DESCRIPTION = "nodeJS Evented I/O for V8 JavaScript"
    PR = "r0"
    DEPENDS = "openssl"
    SRC_URI = " \
    http://nodejs.org/dist/node-v${PV}.tar.gz \
    "
    SRC_URI[md5sum] = "907fa1e0a2f1f0c3df5efc97fd05a7d2"
    SRC_URI[sha256sum] = "a1c776f44bc07305dc0e56df17cc3260eaafa0394c3b06c27448ad85bec272df"
    S = "${WORKDIR}/node-v${PV}"
    do_configure () {
    ./configure
    }
    do_qa_configure () {
    # skip false alarm
    # ${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/node-v0.1.104/build/config.log
    }
    do_compile () {
    make
    }
    do_install () {
    install -d ${D}${bindir}/
    install -m 0755 ${S}/node ${D}${bindir}/
    }
    FILES_${PN} = "${bindir}/node"


node-static_0.1.104.bb
-----------------

Same as above with one major change:

    DEPENDS = "openssl-static"

./node-v0.1.104/deps/libev/wscript
---------------------

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

and another snippet:

    -  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static snapshot=on'
    +  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static'




./node-v0.1.104/wscript
---------------------

    --- a/wscript
    +++ b/wscript
    @@ -319,11 +319,15 @@ def v8_cmd(bld, variant):
       if bld.env['DEST_CPU'] == 'x86_64':
         arch = "arch=x64"
     
    +  if bld.env['DEST_CPU'] == 'arm':
    +    arch = "arch=arm"
    +  
       if variant == "default":
         mode = "release"
       else:
         mode = "debug"
