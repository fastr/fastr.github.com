---
layout: article
title: Using an ssh public key with git team members
categories: ssh team git
updated_at: 2010-08-12
---
Goal
====

A copy-paste tutorial for getting team-members to use their own ssh keys for a project

Note
----

The following are examples. You should change them.

  * `git.example.com:22` Your real domain might be something more like `acmecorp.com` or `74.125.224.16:8022`.
  * `74.125.224.16:8022` Your real ip and port will be different.
  * `tom` Your real user name will be different - perhaps `jerry`.
  * `foobar.git` Your real project might be something like `omap-kernel.git`

The following are not examples.

  * `gitosis` is the user name you must use for cloning a private (internal) git repository.

gitosis users
=============

Create your own ssh key rather than continuing to share a single key with multiple people.

    ssh-keygen -f ~/.ssh/id_rsa -N ''
    # confirm and continue

Move that new key to a temporary shared location

    KEY=~/.ssh/id_rsa.pub
    KEYNAME=`cat ${KEY} | cut -d'=' -f3 | cut -d' ' -f2`
    echo "This new key '${KEYNAME}' should be in your own user name"
    mkdir -p /tmp/keydir/
    cp ${KEY} /tmp/keydir/${KEYNAME}.pub

If you are asked for your **password** when checking out a **git project**, you have not been added correctly and **will not have access**.
Be sure to use `gitosis` as the repository user and not your normal user name.

    # Good Example - using `gitosis`
    git clone gitosis@git.example.com:22/foobar.git
    
    # Bad Example - using your user name
    git clone tom@git.example.com:22/foobar.git

Note that after generating the new key you will be prompted for passwords **in other places** that you may not have been prompted before.

    ssh -p 22 tom@74.125.224.16
    tom@74.125.224.16's password:

You should use `ssh-copy-id` so that you are not asked for your password. This is **more secure** than using a password.

    ssh-copy-id tom@74.125.224.16

If the server is on a special port (not 22) you should edit your `~/.ssh/config`

    vim ~/.ssh/config

`~/.ssh/config`

    Host 74.125.224.16
    User tom
    Port 8022

    Host development-unit6
    User root


git admin
=========

The git admin will add your key to gitosis

    cd ~/gitosis-admin
    mv /tmp/keydir/* ./keydir/
    ls ./keydir/ | while read NAME; do echo ${NAME}; done

And give you access to the projects you need

    cd ~/gitosis-admin
    vim gitosis.conf

More detailed information on setting up gitosis projects is in the previous `git` article.
