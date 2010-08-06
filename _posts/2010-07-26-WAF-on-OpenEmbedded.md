---
layout: article
title: WAF on OpenEmbedded
categories: uncategorized
updated_at: 2010-07-26
---

**Update**: Jul 27th 2010: Works!

desired result
==============

bitbake hello-world-waf

~/overo-oe/tmp/deploy/glibc/ipk/overo/hello-world-waf.ipk

waf.bbclass
-----------

~/overo-oe/user.collection/classes/waf.bbclass

modeled after

~/overo-oe/org.openembedded.dev/classes/autotools.bbclass

    # WAF project class.

    DEPENDS += python

    # Conditionally set the verbose option.
    # WAF_VERBOSE="-v"

    waf_do_compile() {
      ./waf ${WAF_VERBOSE} build 
    }

    waf_do_clean() {
      ./waf ${WAF_VERBOSE} clean
    }

    waf_do_configure() {
      if [ -x ${S}/waf ]; then
        ${S}/waf ${WAF_VERBOSE} configure --prefix=${prefix} ${EXTRA_OECONF} "$@"
      else
        oefatal "executable waf script not found"
      fi
    }

    waf_do_install() {
      ./waf ${WAF_VERBOSE} install --destdir="${D}" "$@"
    }

    EXPORT_FUNCTIONS do_compile do_clean do_configure do_install 

hello-world-waf.bb
----------

~/overo-oe/user.collection/recipes/hello-world-waf/hello-world-waf_1.0.bb

    DESCRIPTION = "Hello World (bitbake remix) feat. WAF"
    PR = "r0"
    DEPENDS = ""
    LICENSE = "GPL"
    SRC_URI = " \
    file://hello-world-waf.c \
    file://wscript \
    file://waf \
    "
    S = "${WORKDIR}"
    FILES_${PN} = "${bindir}/hello-waf"
    
    inherit waf


hello-world-waf.c 
-------------

~/overo-oe/user.collection/recipes/hello-world-waf/files/hello-world-waf.c

    #include <stdio.h>

    int main (int argc, char** argv) {
      printf("Hello World!\n");
      return 0;
    }


wscript 
-------

~/overo-oe/user.collection/recipes/hello-world-waf/files/wscript

    top = '.'
    out = 'build'

    def set_options(opt):
        opt.tool_options('compiler_cc')
      
    def configure(conf):
        conf.check_tool('compiler_cc')

    def build(bld):
        main = bld.new_task_gen()
        main.features = 'cc cprogram'
        main.source = 'hello-world-waf.c'
        main.target = 'hello-waf'
        main.install_path = '${PREFIX}/bin'
        main.chmod = 0755

waf
---

    wget http://waf.googlecode.com/files/waf-1.5.18
    chmod a+x waf-1.5.8
    cp -a waf-1.5.8 ~/overo-oe/user.collection/recipes/hello-world-waf/files/waf


Resources
---------

  * [[PATCH] Add a waf bitbake class. (Koen Kooi)](http://www.mail-archive.com/openembedded-devel@lists.openembedded.org/msg07304.html)
  * [original waf.bbclass](http://www.mail-archive.com/openembedded-devel@lists.openembedded.org/msg07304/waf.bbclass)
