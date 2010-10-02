---
layout: article
uuid: aedce44f-7ca4-4288-87d6-72102a2350c1
title: Sharing a private git repository with a consultant
categories: git team consultant
updated_at: 2010-07-29
---

Goal
====

Allow a consultant to access the source-code of a specific project for a limited time.

Background
----------

Many consultants are hardware gurus or software gurus, but don't necessarily have proficiency with Linux, git, etc. 

Pre-Requisites
--------------

  * Ubunut Server 9.04+
  * gitosis (see previous article)

Informing the consultant
========================

> Tom,
>
> I'm giving you access to our codebase directly so that you can make changes
> and we can get frequent updates and review the changes (and see the comments).
>
> I need you to e-mail me your public ssh key in order to give you access.
> You can find that in `~/.ssh/id_rsa.pub` (or create it using `ssh-keygen`).
>
> Alternatively, you can place the attached `id_rsa` in `~/.ssh/` and start right away.
>
> Once we have the ssh keys you can access the repository like so:
>
>     sudo apt-get install git-core
>     cd ~/
>     git clone ssh://gitosis@git.example.com:22/project-name.git
>
> That will create the `project-name` folder in your home directory.
>
> Once you make changes you will need to need to add, commit, and then push them back to us.
>
>     git stat
>     git add file1 file2 file3
>     git commit -m "some helpful message"
>     git push origin master
>
> You can make as many commits before the push as you'd like.
> The comments will help us as we look it over and test against it.
>
> Thanks for you help on the project!
>
> Nick Burns


Authorizing the consultant
==========================

Create a new private/public key pair from a guest-ish account and e-mail the pair to the consultant

    ssh-keygen
    cp ~/.ssh/id_rsa.pub /tmp/

Add the consultant to gitosis
 
    git clone ssh://gitosis@git.example.com:22/gitosis-admin.git
    cd gitosis-admin
    KEY=/tmp/id_rsa.pub
    KEYNAME=`cat ${KEY} | cut -d'=' -f3 | cut -d' ' -f2`
    echo ${KEYNAME}
    mv ${KEY} keydir/${KEYNAME}.pub
    vim gitosis.conf

Add the consultant to the particular projects

    [group project-name-team]
    writable = project-module-1 project-module-2
    members = keyname-existing-1 keyname-existing-2 consultant-new-1

Then commit the change and the consultant has access

    git add gitosis.conf keydir
    git commit -m "added tom as consultant on project-name"
    git push origin master

Mark your calendar to delete this key after the contract period.
