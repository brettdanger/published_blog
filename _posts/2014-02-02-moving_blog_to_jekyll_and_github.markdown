---
layout: post
title:  "Migrating  My Blog to Jekyll and Github"
date:   2014-02-02 10:34:22
categories: programming
tags: [github, jekyll]
description: "I am migrating my blog to jekyll and github pages.  It is cheaper than my VPS and I only need a server for minor tinkering, which can be done on Amazon for free."
teaser: "http://jekyllrb.com/img/logo-2x.png"
---

My VPS service from myhosting.com is up for renewal, and honestly it isn't worth the money anymore.  I only use it for [my blog](http://www.brettdangerfield.com). I used to host other websites and my private github repos, but now I have a paid github account and no longer host other websites.  I can also spin up an Amazon server if I need to play with some code, otherwise I will code on my own boxes. Github will host static html sites for free and allows for custom domains. The problem is a blog is dynamic, but that is where [Jekyll](http://http://jekyllrb.com) comes in. 

###Jekyll
__Jekyll__ will allow me to program a dynamic site and then it will generate all the static pages for me.

``` ruby
require 'redcarpet'
markdown = Redcarpet.new("Hello World!")
puts markdown.to_html
```