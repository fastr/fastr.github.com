---
layout: article
title: Documenting with Jekyll
categories: meta-documentation
updated_at: 2010-08-02
author: AJ ONeal
---

Goal
====

[Install Jekyll](http://blog.envylabs.com/2009/08/publishing-a-blog-with-github-pages-and-jekyll/) to use for writing documentation in Markdown syntax with code highlighting.

Create a site which has a list of article categories. See [example](http://fastr.github.com/)

Known Issues
-------

This document doesn't describe how to address the issue of **relative paths** of stylesheets, javascripts, etc in `_layouts/default.html`.

Putting Jekyll to Good Use
=========

Installing Jekyll
---------

As per the [documentation](http://wiki.github.com/mojombo/jekyll/install)

    sudo gem install jekyll
    sudo gem install rdiscount
    #sudo apt-get install python-pygments

Because pygments requires extra syntax, I prefer to use [`highlight.js`](http://softwaremaniacs.org/soft/highlight/en/download/) instead. For my needs, highlighting for `Bash`, `C++`, `CSS`, `HTML`, and `JavaScript` suffices. I make the assumption that `highlight.zip` is downloaded to `~/Downloads/highlight.zip`. 

I also recommend `showdown.js` rather than `rdiscount`, but `rdiscount` works well enough and has the (dis)advantage of rendering server-side.

Configuring Jekyll
---------

The most important information can be found here at the [Jekyll wiki](http://wiki.github.com/mojombo/jekyll/template-data)

Creating a home for our site:

    cd ~/
    mkdir blog

Create a site style:

    cd ~/blog
    mkdir images/
    wget http://fastr.github.com/images/ribbedbg.png \
      -O images/ribbedbg.png
    mkdir stylesheets/

`./stylesheets/fastr.css`

    body &#123;
      background-color: #789;
      background-image:url('/images/ribbedbg.png');
    }
    a &#123;
      color: inherit;
      text-decoration: none;
      border-bottom:1px dotted;
      padding-left: 2px;
      padding-right: 2px;
      text-shadow: 1px 1px 1px #888;
      -webkit-text-shadow: 1px 1px 1px #888;
      -moz-text-shadow: 1px 1px 1px #888;
    }
    a:hover &#123;
      border-bottom: 1px solid;
    }
    #article, #header &#123;
      width: 950px;
      background-color: #FFF;
      padding: 10px;
      padding-left: 15px;
      margin: 15px;
      border-radius: 5px;
      -moz-border-radius: 5px;
      -webkit-border-radius: 5px;
    }
    pre &#123;
      width: 900px;
      padding: 5px;
      padding-right: 10px;
      padding-left: 10px;
      margin: 5px;
      margin-top: 20px;
      margin-bottom: 30px;
      box-shadow: 5px 5px 5px #888;
      -webkit-box-shadow: 5px 5px 5px #888;
      -moz-box-shadow: 5px 5px 5px #888;
      border-radius: 5px;
      -moz-border-radius: 5px;
      -webkit-border-radius: 5px;
    }


Using `highlight.js` rather than pygments:

    cd ~/blog
    mkdir vendor/
    # highlight.js as mentioned above
    mv ~/Downloads/highlight.zip vendor/ 
    cd vendor/
    unzip highlight.zip
    rm highlight.zip


Create a site template:

    cd ~/blog
    mkdir _layouts

`./_layouts/default.html`

    <!DOCTYPE html>
    <html>
    <head>
    <title>&#123;&#123; page.title }}</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/fastr.css.css" media="screen" />
    <link rel="stylesheet" type="text/css" href="/vendor/highlight/styles/sunburst.css" media="screen" />
    <script src="/vendor/highlight/highlight.pack.js"></script>
    <script>
      hljs.initHighlightingOnLoad();
    </script>
    </head>

    <body>
      <h1 id="header">&#123;&#123; page.title }}</h1>

      <div id="article">
        &#123;&#123; content }}
      </div>
    </body>
    </html>

`./_layouts/article.html`

    ---
    layout: default
    ---
    <div>
      &#123;&#123; content }}
    </div>
    <em>Updated at &#123;&#123; page.updated_at }} </em>

This next part is a bit of a hack, but it will list all of the categories and also each article by category.

`./index.html`

Note: In the ruby "underneath the hood" (TM) categories is an iterator which returns two elements.
The first is the category name. The second is an array of posts.
Since `liquid` (part of `Jekyll`) doesn't consider the string an iterator, it skips over it in the posts loop.
Hence `for posts in category` only loops over the second argument.
The same affect [can be acheived with `rake`](http://gist.github.com/143571), but that won't run on github.
If you care to [dig into `liquid`](http://wiki.github.com/tobi/liquid/liquid-for-designers), there are alternate ways to solve this problem.

    ---
    layout: default
    title: Fastr Blog
    ---
    <h2>Categories:</h2>
    <ul>
    &#123;% for category in site.categories %}
      <li><a href="#&#123;&#123; category | first }}">&#123;&#123; category | first }}</a></li>
    &#123;% endfor %}
    </ul>

    <h2>Articles by Category:</h2>
    <ul>
    &#123;% for category in site.categories %}
      <li><a name="&#123;&#123; category | first }}">&#123;&#123; category | first }}</a>
        <ul>
        &#123;% for posts in category %}
          &#123;% for post in posts %}
            <li><a href="&#123;&#123; post.url }}">&#123;&#123; post.title }}</a></li>
          &#123;% endfor %}
        &#123;% endfor %}
        </ul>
        <br/>
      </li>
    &#123;% endfor %}
    </ul>

    &#123;% for post in site.categories.quickstart %}
    <!-- h2><a href=".&#123;&#123; post.url }}">&#123;&#123; post.title }}</a></h2 -->
    <!-- &#123;&#123; post.content }} -->
    &#123;% endfor %}

    Page generated: &#123;&#123; site.time | date_to_string }}

Script to create articles:

Articles must be in the format `YYYY-MM-DD-Title-of-Article.markup` such as `2010-08-05-Installing-Jekyll.markdown`.
This script puts the date and format for you, that way you just write the title.

`./mkdocument`

    #!/bin/bash
    TITLE=$&#123;1}
    POSTDIR=./_posts

    if [ ! -n "$&#123;TITLE}" ]; then
      echo "USAGE: mkpost title-of-post"
      exit 1
    fi

    if [ ! -n "$&#123;EDITOR}" ]; then
      if [ -n "`which vim`" ]; then
        EDITOR=vim
      elif [ -n "`which nano`" ]; then
        EDITOR=nano
      elif [ -n "`which emacs`" ]; then
        EDITOR=emacs
      else
        echo "Couldn't find an editor. Tried vim, nano, & emacs. Try \`export EDITOR=your-favorite-editor\` "
      fi
    fi

    FILE=`date "+%Y-%m-%d"`-$&#123;TITLE}.markdown

    if [ ! -e "$&#123;POSTDIR}/$&#123;FILE}" ]
    then
      mkdir -p $&#123;POSTDIR}
      cat - > $&#123;POSTDIR}/$&#123;FILE} << EOF
    ---
    layout: article
    title: `echo $&#123;TITLE} | sed 's/-/ /g'`
    categories: !!UPDATED ME!!
    updated_at: `date +'%Y-%m-%d'`
    rendered: site.time
    ---


    You just created " $&#123;POSTDIR}/$&#123;FILE} "! 

    Notice the UPDATED ME in the categories above.
    Please change that to be a category.

    Your article starts after the last -- above.
    

    Remember
        
        Code blocks are indented by 4 spaces

    Paragraphs have two spaces between lines.
    Sentances have one.

      * lists can be bullet
      * like this

    or

      1. can be numbered
      2. like this

    Large Header
    ====

    Small Header
    ----

    >  block quotes have
    >  a carret and two spaces
    >
    >    and can contain code
    >
    >  * bullets
    >
    >  1. etc
    
    EOF
    fi

    $EDITOR $&#123;POSTDIR}/$&#123;FILE}

Creating an article:

    cd ~/blog
    ./mkpost Name-of-Article
    # new file is created in ~/blog/_posts/

Formating an article:

    ---
    layout: default
    title: Awesome Post
    categories: foo, bar, baz
    ---

    This will be the best blog ever.


Configure our site:

  * `pygments` provides code highlighting
  * `rdiscount` parses markdown formatting more similarly to `showdown`

`./blog/_config.yml`

    destination: /var/www/your_folder_goes_here
    pygments: true
    markdown: rdiscount
    lsi: true
    exclude: mkarticle
    permalink: /articles/:title.html

Hosting the site on Github
====================

Create an account
------------

Jekyll sites can be hosted via [Github Pages](http://pages.github.com/)

  1. visit github.com
  2. click `Signup`
  3. create account (we assume the user `fastr` for this demo)

Add an ssh key
------------

  1. visit your [Account Page](https://github.com/account)
  2. click `Add another public key`
  3. the Title may be any name you wish
  4. `cat ~/.ssh/id_rsaa.pub || ( ssh-keygen && cat ~/.ssh/id_rsa.pub )`
  5. past the output from the line above into the Key

Setup Git
--------

    sudo apt-get install git-core
    git config --global user.name "Your Name Goes Here"
    git config --global user.email your_email_goes_here@email.com

Add the site repository
-------------

  1. visit github.com
  2. click `New repository`
  3. Project Details
    * Name would be `fastr.github.com`

Create the repository
--------------

    cd ~/blog # do this instead of `mkdir fastr.github.com`; cd fastr.github.com
    git init
    touch README.md
    git add .
    git commit -m 'first commit'
    git remote add origin git@github.com:fastr/fastr.github.com.git
    git push origin master
      

Appendix
========

Tags
----

http://gist.github.com/143571

`generate_tags.rb`:

    namespace :tags do
      task :generate do
        puts 'Generating tags...'
        require 'rubygems'
        require 'jekyll'
        include Jekyll::Filters

        options = Jekyll.configuration(&#123;})
        site = Jekyll::Site.new(options)
        site.read_posts('')

        html =<<-HTML
    ---
    layout: default
    title: Tags
    ---

    <h2>Tags</h2>

        HTML

        site.categories.sort.each do |category, posts|
          html << <<-HTML
          <h3 id="#&#123;category}">#&#123;category}</h3>
          HTML

          html << '<ul class="posts">'
          posts.each do |post|
            post_data = post.to_liquid
            html << <<-HTML
              <li>
                <div>#&#123;date_to_string post.date}</div>
                <a href="#&#123;post.url}">#&#123;post_data['title']}</a>
              </li>
            HTML
          end
          html << '</ul>'
        end

        File.open('tags.html', 'w+') do |file|
          file.puts html
        end

        puts 'Done.'
      end
    end


Official Ruby
===========

    To keep things simple, it's probably best to use the official version of ruby and rubygems rather than the pre-packaged ubuntu version.

    If you have Ubuntu 10.10, using the prepackaged version should be fine, but there are several issues with earlier versions and ruby1.8 vs ruby1.9 yuckiness.

I would **always** recommend the official version of rubygems over the pre-packaged version.

Installing GCC (to build ruby)
----------

    sudo apt-get install build-essential

Installing Ruby
----------

Just grabbing the latest release which passes the test cases should be fine.

Just google 'ruby download' or follow my instructions blindly.

    cd ~/
    wget ftp://ftp.ruby-lang.org//pub/ruby/ruby-1.9-stable.tar.gz
    tar xf ruby*
    cd ruby-*
    ./configure && make && make test
    sudo make install

Installing RubyGems
-----------

Easy enough to install. Just google 'rubygems download' or follow my instructions blindly.

    cd ~/
    wget http://production.cf.rubygems.org/rubygems/rubygems-1.3.7.tgz
    tar xf rubygems-*
    cd rubygems-*
    sudo ruby setup.rb
