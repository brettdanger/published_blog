---
layout: post
title:  "Migrating  My Blog to Jekyll and Github"
date:   2014-02-02 10:34:22
categories: programming
tags: [github, jekyll]
description: "I am migrating my blog to jekyll and github pages.  It is cheaper than my VPS and I only need a server for minor tinkering, which can be done on Amazon for free."
teaser: "http://jekyllrb.com/img/logo-2x.png"
---

My VPS service from myhosting.com is up for renewal, and honestly it isn't worth the money anymore.  I only use it for [my blog](http://www.brettdangerfield.com), and it has been a year since I have posted anything. I also used to host other websites and my private github repos, but now I have a paid github account and no longer host those websites. I can also spin up an Amazon server if I need to play with around with the cloud. Github will host static html sites for free and allows for custom domains. The problem is a blog is dynamic, but that is where [Jekyll](http://http://jekyllrb.com) comes in. 

<!--more-->

<a href="http://jekyllrb.com"><img src="/static/img/jekyll_logo_white.png" alt="jekyllrb.com" width="25%"></a><br>
__Jekyll__ is a static html generator that allows me to build a small dynamic site like a blog and generate all the possible html pages.  With [github pages](http://pages.github.com/) I can host my blog for free. It was extremely easy to migrate my site over as I was using the Jinja2 templating system and Jekyll's liquid templating system has darn near identical syntax. I believe Liquid is based off the Django template system, which is very similar to Jinja2.  Once I had the site design migrated I then exported all of my posts and saved them as html.  I simply added them to the `_posts` folder as individual html files.  I had to add some very simple YAML to the top of each post for meta data like dates and tags.

 ```yaml
layout: post
title:  "Migrating  My Blog to Jekyll and Github"
date:   2014-02-02 10:34:22
categories: programming
tags: [github, jekyll]
description: "I am migrating my blog to jekyll and github pages.  It is cheaper than my VPS and I only need a server for minor tinkering, which can be done on Amazon for free."
```

 The biggest challenge was the way I implemented tags and categories on my blog couldn't be duplicated in Jekyll out of the box. Luckily, I found a plugin that adds both categories and tags and enabled me to keep those features. I was also able to keep all my urls in the same format, so I won't have to mess with setting up 301 redirects! I used [disqus](http://www.disqus.com) at the start, so comments were easy to migrate as well.

I should be able to blog more now that I just have to fire up my text editor and commit it to git.  When I originally build my blog, I created a very simple admin page then found I was constantly tinkering with it instead of actually creating new entries.  If you want to see the final product, here is the [repo](http://www.github.com/brettdanger/published_blog) for my jekyll site.  There is a ton of info already out there on using Jekyll and Github pages for blogging, but if you have any questions I would be happy to help. 

