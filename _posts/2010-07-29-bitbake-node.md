---
layout: article
uuid: 58d8a53c-50f8-4cd9-88c9-55a612bc042b
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

  0. [setup your gumstix build environment](http://www.gumstix.net/Setup-and-Programming/view/Overo-Setup-and-Programming/Setting-up-a-build-environment/111.html)
  1. Copy the recipe below and put it in the location specified, creating folders as necessary.
  2. run `bitbake node`
  3. once it fails, go in `${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/node-0.1.104/` and edit the files by hand.
  4. `bitbake node` again. It should succeed.
  5. copy the raw `${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/node-v0.1.104/node` whereever you need it.

Files
=====

These are the files I have so far. They still need a few tweaks before I create a bitbake recipe.

They do compile and leave the useable `node` binary in `${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/node-v0.1.104/node`.

They don't create the `${OVEROTOP}/tmp/deploy/glibc/ipk/armv7a/node_0.1.104.ipk` that you would hope for... yet.

`${OVEROTOP}/user.collection/node/node_0.1.104.bb`
-----------------

    DESCRIPTION = "nodeJS Evented I/O for V8 JavaScript"
    PR = "r0"
    DEPENDS = "openssl"
    SRC_URI = " \
    http://nodejs.org/dist/node-v${PV}.tar.gz \
    "
    #file://libev-arm-crass.patch;apply=yes \
    #file://node-arm-cross.patch;apply=yes \
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


`${OVEROTOP}/user.collection/node/node-static_0.1.104.bb`
-----------------

Same as above with one major change:

    DEPENDS = "openssl-static"

`./node-v0.1.104/deps/libev/wscript` in the temp builddir
---------------------

This will become `files/libev-arm-cross.patch` once packaged.

Currently you can modify the failed build in place from `${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/`

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

`./node-v0.1.104/wscript` in the temp builddir
---------------------

This will become `files/node-arm-cross.patch` once packaged.

Currently you can modify the failed build in place from `${OVEROTOP}/tmp/work/armv7a-angstrom-linux-gnueabi/node-0.1.104-r0/`

    --- a/wscript
    +++ b/wscript
    @@ -319,11 +319,15 @@ def v8_cmd(bld, variant):
       if bld.env['DEST_CPU'] == 'x86_64':
         arch = "arch=x64"
     
    +  if bld.env['AR'] == 'arm-angstrom-linux-gnueabi-ar': # TODO use -1 != str.find('arm-xxx-linux-gnueabi)
    +    arch = "arch=arm"
    +  
       if variant == "default":
         mode = "release"
       else:
         mode = "debug"

and another snippet:

    -  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static snapshot=on'
    +  cmd_R = 'python "%s" -j %d -C "%s" -Y "%s" visibility=default mode=%s %s library=static'

