---
layout: article
title: Troubleshooting bitbake
categories: bitbake
updated_at: 2010-08-14
---
Goal
====

Aleviate some of the `bitbake` headaches.

  * NOTE: `${OVEROTOP}` refers to the **actual environment variable**
  * NOTE: `${PACKAGE}` should be replaced with the **name of the package** you are trying to build.

Quick hacking without `devshell`
-------

If the package that you're building doesn't build you can intercept manually build in the bitbake environment and then continue.

  0. `bitbake ${PACKAGE}`
  1. cd `${OVEROTOP}/tmp/work/${ARCH}/${PACKAGE}_${VER}_r${REV}/${BUILD}`
    * `${ARCH}` is probably `armv7a-angstrom-linux-gnueabi`
    * `${BUILD}` is a directory which is not NOT `src` or `temp` such as the package name or `git` or `trunk` or `svn`
    * `${VER}` and `${REV}` - duh.
  2. `cp ../temp/run.do_compile.${OLD_PID} ./`
    * `${OLD_PID}` is a number like `3597` or `4352`
    * `do_compile` could be any task - `do_install`, etc
  3. `vim run.do_compile.1234`
    1. comment out the last line: `do_compile()` (or `do_install()` or whatever)
    2. add `bash --norc`
  4. `./run.do_compile.${OLD_PID}` will put you in an environment with all variables set
  5. `make` (or whatever) to try to build, debug issues
  6. `exit` (when done to go back to shell without run.do_compile settings)
  7. `bitbake ${PACKAGE}`

devshell
========

[`devshell`](http://wiki.openembedded.net/index.php/Devshell) executes after `do_patch()` and will give you a shell with the environment variables you need for building the package.

Common Problems
=====================

`bitbake` is a headache. That's just the way it is. Sorry. It's a moving target, there are lots of changes committed every few days. The testing is perhaps substandard... but it is fast-moving.

**`bitbake -c clean ${PACKAGE}`** often - after each new thing you try in the trouble shooting procedure.

Be very careful to read the error messages. Many of these errors are result of simple copy/paste/delete typos and "could not find" xzy is simply a missing symlink or a file in the wrong place.

There are a number of problems that I run into over and over.

`/bin/sh couldn't find xyz`
---------

Possibilities

NOTE: `xyz` refers to some other package, there is no package named `xyz`.

  * `bitbake xyz-native` - it may just be that the package maintainer left `DEPENDS += "xyz-native"` out of xyz-native_0.bb
    1. `bitbake xyz-native
    2. `bitbake -c clean ${PACKAGE}`
    3. exit the current shell or terminal (to reset environment variables) and reopen a new one
    4. `bitbake ${PACKAGE}`

  * xyz is a broken symlink
    * find ${OVEROTOP}/tmp/ -name '*xyz*'

  * No, really, exit the current shell and reopen it. Maybe even reboot? (I've never had to do this)

`md5sum mismatch`
-------

  * the download had a hiccup
    1. `rm sources/${PACKAGE}*`
    2. `bitbake -c clean ${PACKAGE}; bitbake ${PACKAGE}`

  * the download script can't get the file correctly
    1. look in `${OVERTOP}/org.openembedded.dev/recipes/${PACKAGE}_${VER}.bb`
    2. `wget http://example.com/xyz.tgz`
    3. mv xyz.tgz ${OVEROTOP}/sources/
    4. `bitbake -c clean ${PACKAGE}; bitbake ${PACKAGE}` 

  * the file has been updated, but the md5sum in `${PACKAGE}_${VER}.bb` hasn't.
    1. follow steps 1 and 2 above
    2. md5sum xyz.tgz > sources/xyz.tgz.md5
    2. sha2sum xyz.tgz > sources/xyz.tgz.sha256
    3. add the checksums to `${OVERTOP}/org.openembedded.dev/recipes/${PACKAGE}_${VER}.bb`
    4. `bitbake ${PACKAGE}`

couldn't download the package
-------

Some commercially available apps (such as TI's dsplink) have bitbake packages, but you must manually download the source for various legal reasons.
    
Worst Case
=========

Save your changes to `user.collections` and start afresh.
If you're super lucky you can just rebuild everything over the next few days and not have to keep solving failed build problems.
In the worst worst case you might want to wait a week for patches to come in.

    rm ${OVEROTOP}/tmp -rf

or even 

    mkdir ${OVEROTOP}.bak
    mv ${OVEROTOP}/user.collection ${OVEROTOP}.bak/
    rm ${OVEROTOP} -rf

and just start from the gumstix documentation for installation all over again


Some of these worst case fixes are required when

  * ABI has changed
  * tmp directory has moved

