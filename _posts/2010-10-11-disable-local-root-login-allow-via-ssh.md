---
layout: article
uuid: 374ff41c-440b-42a9-b095-37777087040d
title: Backup / Clone via ssh (root login disabled)
name: backup-clone-via-ssh-without-root-login
created_at: 2010-10-11
updated_at: 2010-10-11
categories: policy
---

Goal
====

I want to use rsync to clone/backup a server such that if the main server goes down (or I accidentally `rm -rf /usr` instead of `./usr` again) 
the recovery time should be in the order of minutes.

ssh key-only login for root
====

This doesn't allow root to login with a password, only with a key.

    sudo su -
    ssh-keygen -f .ssh/id_rsa -P ''
    cat .ssh/id_rsa.pub >> .ssh/authorized_keys

`/etc/ssh/sshd_config`

    PermitRootLogin without-password
    PermitEmptyPasswords no

rysnc backups
====

Both servers have the exact same hard drive partitioning scheme and files - minus the UUIDs, network configuration, and backups folder.

Of course, I don't want to backup 

  * devcie specific configuration: grub UUIDs, network interfaces, `/boot`
  * virtual filesystems: `sys`, `dev`, `proc`, `.gvfs`
  * temporary filesystems: `/tmp`, `/var/run`, `/media`
  * SCM repositories: `.git`, `.svn`

`/etc/cron.daily/backup`:

    #!/bin/sh

    CLONE='clone.example.tld'
    /usr/bin/rsync -avh / ${CLONE}:/ \
          --exclude=/dev \
          --exclude=/proc \
          --exclude=/sys \
          --exclude=/media \
          --exclude=/tmp \
          --exclude=cache \
          --exclude=/etc/fstab \
          --exclude=/boot/grub \
          --exclude=/var/run \
          --exclude=/etc/hostname \
          --exclude=/etc/cron.daily/backup \
          --exclude=/etc/network/interfaces \
          --exclude='.git' \
          --exclude='.svn' \
          --exclude='.gvfs' \
          --exclude=/mnt/local/backup/ \
          --backup \
          --backup-dir=/mnt/local/backup/`date '+%F_%H-%M-%S'` \
          --delete


Appendix
====

Resources:

  * http://serverfault.com/questions/111187/disable-local-root-login-permit-root-login-over-ssh

`man sshd_config`

     PermitRootLogin
             Specifies whether root can log in using ssh(1).  The argument
             must be ``yes'', ``without-password'', ``forced-commands-only'',
             or ``no''.  The default is ``yes''.

             If this option is set to ``without-password'', password authenti-
             cation is disabled for root.

             If this option is set to ``forced-commands-only'', root login
             with public key authentication will be allowed, but only if the
             command option has been specified (which may be useful for taking
             remote backups even if root login is normally not allowed).  All
             other authentication methods are disabled for root.

             If this option is set to ``no'', root is not allowed to log in.
