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

    GUMSTIX=192.168.1.110
    scp -r /usr/local/lib/node/waf* ${GUMSTIX}:/usr/lib/node/
    ssh ${GUMSTIX}
    sudo opkg install git task-native-sdk python python-modules python-distutils python-misc python-subprocess

Installing node-inotify

    git clone 
    cd node-inotify
    node-waf configure build

NOTE: The python modules may not all need to be installed. I'm unsure.

Appendix
====

Errors you get without the `wafadmin` tools:

    node-waf configure build
    Traceback (most recent call last):
      File "/usr/bin/node-waf", line 14, in <module>
        import Scripting
    ImportError: No module named Scripting

Errors you get without the `python-subprocess` module:

    node-waf configure build
    Traceback (most recent call last):
      File "/usr/bin/node-waf", line 14, in <module>
        import Scripting
      File "/usr/bin/../lib/node/wafadmin/Scripting.py", line 9, in <module>
        import Utils, Configure, Build, Logs, Options, Environment, Task
      File "/usr/bin/../lib/node/wafadmin/Utils.py", line 43, in <module>
        import subprocess as pproc
    ImportError: No module named subprocess
