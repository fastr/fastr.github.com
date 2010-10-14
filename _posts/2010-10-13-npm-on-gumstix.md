---
layout: article
uuid: d0153ba3-b4f3-420f-b943-b4087c189433
title: npm-on-gumstix
name: npm-on-gumstix
created_at: 2010-10-13
updated_at: 2010-10-13
categories: nodejs gumstix
---
Goal
====

Use `npm` to install spark, express, connect on a gumstix overo (like beagleboard).

Install nodejs
====

nodejs is already available in OpenEmbedded

    bitbake nodejs
    TARGET=192.168.1.110
    scp tmp/deplay/glibc/ipk/armv7a/nodejs_*.ipk ${TARGET}:~/
    opkg install nodejs_*.ipk

and should soon be available in Angstrom (if the commit messages on bitbucket indicate anything)

    opkg install nodejs

Making a user
====

    USER=fastr
    GROUP=wheel
    visudo # uncomment %wheel
    addgroup ${GROUP}
    adduser ${USER} ${GROUP}


Create a sandbox
---

    su ${USER}
    cd
    echo 'export PATH=~/local/bin:${PATH}' >> ~/.bash_profile
    . ~/.bash_profile

    mkdir ~/local/bin ~/local/share/man -p

    cat >>~/.npmrc <<NPMRC
    root = $HOME/.node_libraries
    binroot = $HOME/bin
    manroot = $HOME/share/man
    NPMRC

TODO: use list of files to copy node locally also

Installing npm
====

  ntpdate ntp.ubuntu.com
  sudo opkg install curl make
  curl http://npmjs.org/install.sh | sh

Installing a few modules
====

  npm install connect express spark futures couchdb

test
----

    mkdir ~/webapps

`webapps/app.js`:

    var connect = require('connect');

    module.exports = connect.createServer(function (req, res) {
      console.dir(req.headers);
    });

`webapps/config.js`:
    
    module.exports = {
      port: 3000,
      //user: "nobody", // set this when running as root in production
      env: "development"
    }

    cd ~/webapps
    spark

Tada!
