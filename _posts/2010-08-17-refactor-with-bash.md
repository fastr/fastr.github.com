---
layout: article
title: refactor with bash
categories: util
updated_at: 2010-08-17
---

Recursively replaces all occurances of `<old-string>` with `<new-string>` in `<file-or-dir>`.


Note that only whole strings are replaced thanks to `\<` and `\>`:

    A nice text file which isn't tex.

    refactor.sh tex md myfile

    A nice text file which isn't md.

`refactor.sh`
============


    !/bin/bash

    OLD=${1}
    NEW=${2}
    TARGET=${3}

    if [ ! -n "${TARGET}" ]
    then
        echo "USAGE: ${0} <old-string> <new-string> <file-or-directory>"
        exit 1
    fi

    OLD=`echo "${OLD}" | sed 's/\\./\\\\./g'`
    NEW=`echo "${NEW}" | sed 's/\\./\\\\./g'`

    if [ ! -f "${TARGET}" ]
    then
        cd "${TARGET}"
        find ./ -type f | grep -v '.svn' | xargs sed -i "s/\<${OLD}\>/${NEW}/gi"
    else
        sed -i "s/\<${OLD}\>/${NEW}/gi" "${TARGET}"
    fi
