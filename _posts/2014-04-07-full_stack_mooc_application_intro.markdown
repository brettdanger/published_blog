---
layout: post
title:  "Mooc Aggregator Project, Introduction"
date:   2014-04-07 10:34:22
categories: programming
tags: [github, python, angularjs, sqlalchemy, postgres, mongodb, coursera]
description: "I am starting to build a full stack online course aggregator and rating site, but I am going to open it up to github."
teaser: "http://www.mindingthecampus.com/originals/mooc.jpg"
---

MOOCs are [Massive Open Online Courses](http://en.wikipedia.org/wiki/Massive_open_online_course), and I have had the time to take a few of them. These courses have probably been the best college courses I have taken of any kind, in terms of how much I learned and have been able to apply to my day to day work.  This makes sense as I have been able to pick courses that interest me or apply specifically to my job versus my normal course work where the college decides what I need. The other reason they have been great is because they are provided by some of the best universities or professors in that topic.  

<!--more-->

One problem with MOOCs are the fact that there are so many courses available from so many different providers that it is hard to know which one to spend time on.  There are some MOOC aggregators out there that provide listings and ratings across providers, but I haven't found one that stands out as a great experience.  One of my first thoughts was this could be an opportunity to build a business, however to monetize that would initially require me to puke adds all over the page. Ad-heavy sites is one of the main problems I discovered with the existing mooc aggregators.

So instead, I am going build an aggregator and open source the code and see if others can build something better. Building a full stack mooc aggregator site is a perfect blogging topic and a chance to build a public facing code portfolio as almost all of my code is proprietary to my employers. It also gives me several posts worth of topics to write about, which I never seem to think I have something useful to write.

Here is an outline of the project that I will build and write about:

- [Intro to the project (this post)](/post/full_stack_mooc_application_intro/)
- **Part 1** - [Building a python-based backend that collects course listings and stores them. Initially get courses from Coursera and save to MongoDB and an RDMBS (postgres)](/post/full_stack_mooc_app_part_1_python_backend/)
- **Part 2** - Add more providers (EDX) and storage engines (ElasticSearch)
- **Part 3** - Port backend to NodeJS as a learning experience and point of comparison
- **Part 4** - Build an API that will be used to power the website and mobile app. (Flask for python, ExpressJS for Node)
- **Part 5** - Build a frontend website with Flask and ExpressJS, for crawl based site. Public listings and ratings
- **Part 6** - Build Web application for rating courses, saving a personal wish list, and other ideas. Use AngularJS for single page web app
- **Part 7**- Evaluate PAAS service providers like Heroku and Google AppEngine, compare to AWS, Rackspace, Linode
- **Part 8** - Build a mobile site using PhoneGap (cordova)
- **Part 9** - Build a native app for Android and Iphone
- **Part 10** - Report findings/learnings/benchmarks

You can refer to this page as a table of contents as I will turn this outline into links as each topic is completed.  I am hoping to post weekly or every other week (yea right). 

I am calling this **"Project Wisdom"** (yes it's a dumb name but I am not good at naming things*).

<span style="font-size: 7pt">*except my children</span>


## Project Wisdom Code:

**<a class="btn btn-large" href="http://www.github.com/brettdanger"><i class=" icon-github" target="_blank"></i>&nbsp;&nbsp;Wisdom Backend</a>**


