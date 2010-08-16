---
layout: article
title: Set up a private git server on ubuntu
categories: git team
updated_at: 2010-07-29
---

Goal
====

    git clone git://my-private-server.com/project.git

Setup
=====

The server shall be called 'git.example.com'.

The clients shall be called 'The Client'

A note about clients
--------------------

`gitosis` uses ssh keys rather than a password to verify identity.

For each client you will need to create an ssh key (if you don't already have one).

    KEYFILE=~/.ssh/id_rsa
    if [ ! -f "${KEYFILE}" ]
    then
      ssh-keygen -f ${KEYFILE} -N ''
    fi

Each client will also need to have it's public key listed for gitosis

    /srv/gitosis/.ssh/authorized_keys

Getting Started
---------------

To make things simple this will all take place from the perspective of the client machine.

    ssh mike@git.example.com
    apt-get -y install git-core gitosis
    exit
    
    scp ~/.ssh/id_rsa.pub mike@git.example.com:/tmp
    
    ssh mike@git.example.com
    sudo -H -u gitosis gitosis-init < /tmp/id_rsa.pub
    rm /tmp/id_rsa.pub
      # perform this step only if you don't already allow regular users in via ssh
      #   sudo vim /etc/ssh/sshd_config
      #   AllowUsers gitosis
      #   sudo /etc/init.d/sshd restart
    exit
    
    cd ~/ # or wherever you'd like
    git clone ssh://gitosis@git.example.com:22/gitosis-admin.git
    vim gitosis-admin/gitosis.conf

For each member of the project you must add their ssh key. If they were to have e-mailed you their public keys, the process might look like this this:

> Tom, Dick, Harry
>
> Please send me a copy of your ssh key via e-mail or by placing it in /tmp/keys on the server.
>
> 
> Thanks,
> <br/>Nick Burns
> <br/>ACME CTO

    cd ~/
    mv ~/Downloads/id_rsa.pub gitosis-admin/keydir/tom@tom-macbook-pro-17.local.pub
    mv ~/Downloads/id_rsa.pub.1 gitosis-admin/keydir/dick@ubuntu-desktop.pub
    mv ~/Downloads/id_rsa.pub.2 gitosis-admin/keydir/harry@git.example.com.pub

The name of the file must match the name inside, with the .pub extension.

    cat ~/.ssh/id_rsa.pub
    ssh-rsa AAAAB3N[..omitted..]zafWw== coolaj86@ubuntu-server    

An automated version of the process

    ls ~/Downloads/*.pub | while read KEY
    do
      KEYNAME=`cat ${KEY} | cut -d'=' -f3 | cut -d' ' -f2`
      mv ${KEY} gitosis-admin/keydir/${KEYNAME}.pub
    done

Once you have the keyfiles in the right place, you must specify which keyfile has access to which project

    [group team_foobars]
    writable = project_baz project_plusplus
    members = tom@tom-macbook-pro-17.local dick@ubuntu-desktop harry@git.example.com
    description = A collection of fizzbuzz from all employees
    owner = Nick Burns

Now you must commit your changes

    cd ~/gitosis-admin
    git status
    git diff
    git add gitosis.conf keydir
    git commit -m "added project baz and its members"
    git push origin master

Creating a new project
======================

Let's say that `project_baz` (as described above) is a new project. We could add it like so:

    mkdir project_baz
    cd project_baz
    git init
    git remote add origin ssh://gitosis@git.example.com:22/project_baz.git
    
    touch README
    git add .
    git commit -m "Initial import"
    
    git push origin master


Resources
=========

  * [cheatpages/gitosis](http://archive.daniel-baumann.ch/debian/documents/cheatpages/gitosis.html)
  * [gitosis_management](http://www.mantisbt.org/wiki/doku.php/mantisbt:gitosis_management)
  * [Ubunut Community Wiki: Git](https://help.ubuntu.com/community/Git)
