---
layout: article
uuid: 8c1fcee3-3fd1-4604-bb65-4641c5675f30
title: Sharing project files with a team using ACLs on Ubuntu
categories: unfinished ubuntu team
updated_at: 2010-07-29
---

Goal
====

One installation of overo-oe that all team-members can have read-write access to.

Installation
============

The acl tools package

    sudo apt-get install acl

This contains

    acl
    getfacl
    setfacl

The gui acl package for nautilus

    sudo apt-get install eiciel


Mounting with ACLs
==================

Modify the `/` or `/home` partition (or wherever) to include ACL support

    vim /etc/fstab

Just add `acl` after defaults

    # /home was on /dev/sda5 during installation
    UUID=488cd15e-8d2a-45e2-9ed0-00f9643e8cf2 /home           ext3    defaults,acl        0       2    

And then remount

    sudo mount /home -o remount

Setting up a shared location
============================

To keep things simple I create a directory `/home/shared`, owned by root and give all users rwx by default.

This means that any new file or directory created within this directory wil have rwx.

    setfacl --physical --set default:other::rwx /home/shared/
    # or in a more sensitive environment
    # setfacl --recursive --set u:harry:rwx /home/shared

The caveat is that files *copied* or *moved* will have the same permission that they previously had.
They will not get updated.

    cd ~/
    mv overo-oe/tmp /home/shared/overo-oe-tmp
    ln -s /home/shared/overo-oe-tmp overo-oe/tmp
    setfacl --recursive --set default:other::rwx /home/shared/overo-oe-tmp


Resources
=========

  * [acl examples](https://supportweb.cs.bham.ac.uk/information/academic/uacls.php)
  * [setfacl man page](http://gd.tuwien.ac.at/linuxcommand.org/man_pages/setfacl1.html)
  * [ACL Inheritance in Ubuntu](http://ubuntuforums.org/showthread.php?t=691127)
  * [Ubuntu Community Wiki: File Permissions](https://help.ubuntu.com/community/FilePermissions)
  * [Ubuntu Access Control Lists](http://beginlinux.com/server_training/server-managment-topics/1038-ubuntu-804-access-control-lists)
