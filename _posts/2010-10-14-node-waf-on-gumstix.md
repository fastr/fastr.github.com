---
layout: article
uuid: 0c628806-fc13-462d-9500-a14e718c5c0b
title: node-waf on gumstix
name: node-waf-on-gumstix
created_at: 2010-10-14
updated_at: 2010-10-14
categories: nodejs gumstix
---
Goal
====

Install a `node` module natively using `node-waf`.

Specifically, I'm looking at `node-inotify`

Procedure
====

For reasons that I don't understand, the packaged build of `nodejs` in OpenEmbedded doesn't include `node`'s `waf` tools.
I just copied them over from a system that does have them installed:

Prereqs
----

    GUMSTIX=192.168.1.110
    scp -r /usr/local/lib/node/waf* ${GUMSTIX}:/usr/lib/node/
    scp ~/overo-oe/tmp/deploy/glibc/ipk/armv7a/nodejs-dev_*.ipk ${GUMSTIX}:~/
    ssh ${GUMSTIX}
    sudo opkg install nodejs-dev_*.ipk
    sudo opkg install git task-native-sdk python python-modules python-distutils python-misc python-subprocess

Installing node-inotify
----
    git clone git://github.com/c4milo/node-inotify.git
    cd node-inotify
    node-waf configure build
    node-waf install # installs locally

`npm install inotify` has the (dis)advantage of installing to the same location as `node`,
which is in `/usr` in the case of `opkg install nodejs`

NOTE: The python modules may not all need to be installed. I'm unsure.

Test
----

`test-inotify.js`

    


Appendix
====

Errors you get when `/var/volatile` is full:
----

And actually, you get a lot of other miscellaneous errors too.

    sudo opkg install ./python-subprocess_2.6.5-ml12.1.6_armv7a.ipk 
    Collected errors:
     * pkg_init_from_file: Malformed package file ../python-subprocess_2.6.5-ml12.1.6_armv7a.ipk.

Errors you get without the `wafadmin` tools:
----

    node-waf configure build
    Traceback (most recent call last):
      File "/usr/bin/node-waf", line 14, in <module>
        import Scripting
    ImportError: No module named Scripting

Errors you get without the `python-subprocess` module:
----

    node-waf configure build
    Traceback (most recent call last):
      File "/usr/bin/node-waf", line 14, in <module>
        import Scripting
      File "/usr/bin/../lib/node/wafadmin/Scripting.py", line 9, in <module>
        import Utils, Configure, Build, Logs, Options, Environment, Task
      File "/usr/bin/../lib/node/wafadmin/Utils.py", line 43, in <module>
        import subprocess as pproc
    ImportError: No module named subprocess

Errors you get when `nodejs-dev` is not installed:
----

    node-waf configure build
    Checking for program g++ or c++          : /usr/bin/g++ 
    Checking for program cpp                 : /usr/bin/cpp 
    Checking for program ar                  : /usr/bin/ar 
    Checking for program ranlib              : /usr/bin/ranlib 
    Checking for g++                         : ok  
    Checking for node path                   : not found 
    Checking for node prefix                 : ok /usr 
    Checking for program node                : /usr/bin/node 
    Checking for function inotify_init       : yes 
    'configure' finished successfully (2.605s)
    Waf: Entering directory `/home/user/node-inotify/build'
    [1/3] cxx: src/node_inotify.cc -> build/default/src/node_inotify_1.o
    In file included from ../src/bindings.h:4,
                     from ../src/node_inotify.cc:2:
    ../src/node_inotify.h:5:18: error: node.h: No such file or directory
    ../src/node_inotify.h:6:25: error: node_events.h: No such file or directory
    In file included from ../src/bindings.h:4,
                     from ../src/node_inotify.cc:2:
    ../src/node_inotify.h:15: error: 'v8' is not a namespace-name
    ../src/node_inotify.h:15: error: expected namespace-name before ';' token
    ../src/node_inotify.h:16: error: 'node' is not a namespace-name
    ../src/node_inotify.h:16: error: expected namespace-name before ';' token
    In file included from ../src/node_inotify.cc:2:
    ../src/bindings.h:8: error: expected class-name before '{' token
    ../src/bindings.h:10: error: 'Handle' has not been declared
    ../src/bindings.h:10: error: expected ',' or '...' before '<' token
    ../src/bindings.h:16: error: ISO C++ forbids declaration of 'Handle' with no type
    ../src/bindings.h:16: error: expected ';' before '<' token
    ../src/bindings.h:17: error: ISO C++ forbids declaration of 'Handle' with no type
    ../src/bindings.h:17: error: expected ';' before '<' token
    ../src/bindings.h:18: error: ISO C++ forbids declaration of 'Handle' with no type
    ../src/bindings.h:18: error: expected ';' before '<' token
    ../src/bindings.h:19: error: ISO C++ forbids declaration of 'Handle' with no type
    ../src/bindings.h:19: error: expected ';' before '<' token
    ../src/bindings.h:20: error: ISO C++ forbids declaration of 'Handle' with no type
    ../src/bindings.h:20: error: expected ';' before '<' token
    ../src/bindings.h:25: error: 'ev_io' does not name a type
    ../src/bindings.h:27: error: 'EV_P_' has not been declared
    ../src/bindings.h:27: error: expected ',' or '...' before '*' token
    ../src/node_inotify.cc:5: error: variable or field 'InitializeInotify' declared void
    ../src/node_inotify.cc:5: error: 'Handle' was not declared in this scope
    ../src/node_inotify.cc:5: error: 'Object' was not declared in this scope
    ../src/node_inotify.cc:5: error: 'target' was not declared in this scope
    ../src/node_inotify.cc:20: warning: 'init' initialized and declared 'extern'
    ../src/node_inotify.cc:20: error: variable or field 'init' declared void
    ../src/node_inotify.cc:20: error: 'Handle' was not declared in this scope
    ../src/node_inotify.cc:20: error: 'Object' was not declared in this scope
    ../src/node_inotify.cc:20: error: 'target' was not declared in this scope
    Waf: Leaving directory `/home/user/node-inotify/build'
    Build failed:  -> task failed (err #1): 
      {task: cxx node_inotify.cc -> node_inotify_1.o}

