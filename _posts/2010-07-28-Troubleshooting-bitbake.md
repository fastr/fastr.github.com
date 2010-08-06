---
layout: article
title: Troubleshooting bitbake
categories: uncategorized
updated_at: 2010-07-28
---

bitbake is a headache
=====================

That's just the way it is. Sorry. It's a moving target, there are lots of changes committed every few days. The testing is perhaps substandard... but it is fast-moving.

The worst possible thing you can do is to scrap your whole overo-oe folder and start over. That could take days.

My Recommendations
------------------

    cd ~/overo-oe
    bitbake -c clean package-name
    rm sources/<package-name>*
    bitbake package-name


If you encounter more errors, just step through them one-by-one until you've got things fixed. It's long, it's painful, it's the only way... I know of.

Be very careful to read the error messages. Many of these errors are result of simple copy/paste/delete typos and "could not find" xzy is simply a missing symlink or a file in the wrong place.
