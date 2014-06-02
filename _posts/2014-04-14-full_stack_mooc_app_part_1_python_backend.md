---
layout: post
title:  "Mooc Aggregator Project, Python backend with MongoDB and SQLAlachemy"
date:   2014-04-14 10:34:22
categories: programming
popular: true
tags: [github, python, sqlalchemy, postgres, mongodb, coursera]
description: "Python Backend for online course aggregator. Building a modular backend with Python that supports multiple course providers and storage engines. This is part 1 with support for coursera and a storage engine for mongodb and relational databases using SQLAlchemy."
teaser: "/static/img/mooc_python.png"
---
### Part 1 - Creating a python based backend
I decided to build an online course aggregator and introduced the project in [this](/post/full_stack_mooc_application_intro/) post.  This post is part 1 of the series and I will break down how I build the backend aggregation framework.  The whole point of this project is to build a full stack application as both a learning exercise and an in-depth blogging topic.

<!--more-->
The idea for the backend is to collect course data from the major online course providers, normalize the data, and store the data locally.  This aggregator will most likely will run on an interval via cron or maybe supervisord.  I also wanted the code to be modular so it is very easy to add a provider or a storage engine.  The collection methods will very from provider to provider, for instance some may provide listings via an API like [Coursera](https://www.coursera.org/) while others may need to be scraped using something like [beautiful soup](http://www.crummy.com/software/BeautifulSoup/bs4/doc/).

In order to make it modular, I can create a provider base class and a storage engine base class that each module can inherit.  This makes the main program very simple; insert a list of providers and engines and iterate over them. 

``` python
import importlib

provider_list = ["coursera", "edx"]
storage_list = ["mongodb", "postgres"]

#loop over the providers and call their get courses method
for provider in provider_list:
    print "Collecting Courses for provider: {}".format(provider) #use importlib to "polymorphicly call the providers"
    mod = importlib.import_module("providers." + provider.lower()) #import module
    func = getattr(mod, provider)() #create instance
    data = getattr(func, "get_courses")() #call get courses, from any provide we expect the same dictionary structure to be returned

    #loop through Storage Engines Lists
    for store in storage_list:
        print "Saving Courses into Engine: {}".format(store)
        mod = importlib.import_module("storage." + store.lower())
        func = getattr(mod, store)()
        getattr(func, "store_courses")(data) #call the store courses method to save the data

```

This is the entirety of the main program, to collect data from a new provider simply create another provider class and add the name to the provider list. Let's show how to create a new provider.

### Creating Providers

I am starting with [Coursera](http://www/coursera.org) as it is the most popular [mooc](http://en.wikipedia.org/wiki/Massive_open_online_course) provider and it has REST/JSON endpoints that makes it easy to collect the data.  Before we dive into the Coursera driver, let's talk about the generic provider class that each driver will inherit.  Any driver simply needs to provide a get_courses method that returns a normalized dictionary of data. In my generic provider class I created a method that gives that basic dictionary structure.  I used python's [ABC](http://docs.python.org/2/library/abc.html) structure for creating abstract methods.  Here is the basic provider base class.

``` python

from abc import ABCMeta, abstractmethod


class ProviderBase(object):
    __metaclass__ = ABCMeta


    #all provider drivers need to create a get_courses method
    @abstractmethod
    def get_courses(self):
        #abstract implemented in each provider
        pass

```

I also wanted to define the dictionary structure that the storage engines will expect, so I created a static method that defines that dictionary structure.  I actually wanted this to be a private static method only available in the class, so I used the *@classmethod* decorator.

``` python
    @classmethod
    def get_schema_map(cls):
        course_schema = {
            "course_name": None,
            "provider": None,
            "language": None,
            "instructor": None,
            "providers_id": None,
            "media": {
                "photo_url": None,
                "icon_url": None,
                "video_url": None,
                "video_type": None,
                "video_id": None
            },
            "short_description": None,
            "full_description": None,
            "course_url": None,
            "institution": {
                "name": None,
                "description": None,
                "id": None,
                "website": None,
                "logo_url": None,
                "city": None,
                "state": None,
                "country": None
            },
            "sessions": [],
            "workload": None,
            "categories": [],
            "tags": []
        }

        return course_schema
```

### Creating the first driver
So now lets dive into the Coursera driver, this should be pretty simple to implement as I was able to find some REST endpoints in their front end backbone.js code.  The url to get all courses is: *https://www.coursera.org/maestro/api/topic/list?full=1* and to get the detail about a specific course the url is: *https://www.coursera.org/maestro/api/topic/information?topic-id=:id*.  This makes it very simple to collect and normalize. I used the requests library to collect the data. This made the Coursera driver pretty easy to build, however I didn't want to keep hitting the API over and over while I developed the driver, so I used a test driven approach and mocked the [requests](http://docs.python-requests.org/en/latest/) library.  I was able to save the json responses into a file and load them via a mock.

Here is my final coursera driver:

``` python
from provider import ProviderBase
import requests
from datetime import date


class Coursera(ProviderBase):
    def __init__(self):
        self.course_data = []

    def get_courses(self):
        coursera_url = "https://www.coursera.org/maestro/api/topic/list?full=1"

        response = requests.get(coursera_url)
        courses = response.json()
        catalog = []
        for item in courses:
            course = Coursera.get_schema_map()
            print "Processing Course: Coursera - {}".format(item.get("name", "Unknown").encode('utf-8'))
            try:
                #get required items
                course['course_name'] = item['name']
                course['providers_id'] = item["short_name"]
                course['provider'] = "coursera"
                course['language'] = Coursera.get_valid_language(item['language'])
                course['instructor'] = item['instructor']
                course['course_url'] = "http://class.coursera.org/{}/".format(item["short_name"])
                #lets create an ID for the record
                course['id'] = Coursera.create_id(course['provider'] + course["course_name"])
                #get institution data
                university = item['universities'][0]
                institution = {
                    "name": university['name'],
                    "description": university.get("description", None),
                    "id": Coursera.create_id(university['name']),
                    "website": university["home_link"],
                    "logo_url": university["logo"],
                    "city": university["location_city"],
                    "state": university["location_state"],
                    "country": university["location_country"]
                }
                course['institution'] = institution

                #get the data we need from the full course detail
                more_details = self.__get_course_detail(course["providers_id"])

                course['full_description'] = more_details.get("about_the_course", "not found")
            except KeyError:
                #we don't have all required fields, skip for now
                #log it
                continue

            #get MEDIA INFO
            media = {
                "photo_url": more_details.get("photo", None),
                "icon_url": more_details.get("large_icon", None),
                "video_url": more_details.get("video_baseurl", None),
                "video_type": "mp4",
                "video_id": more_details.get("video_id", None)
            }

            course["media"] = media

            #get optional fields
            course['short_description'] = item.get('short_description', None)
            course['categories'] = item.get('categories', [])
            course['workload'] = more_details.get('estimated_class_workload', None)
            catalog.append(item['short_name'])

            #get tags
            tags = []
            for cat in more_details["categories"]:
                tags.append(cat["name"])

            course["tags"] = tags

            #get the session data
            for c in item.get('courses'):
                session = {}
                session['duration'] = c.get('duration_string', None)
                session['provider_session_id'] = c.get('id', None)
                #get Start Date
                if all(name in c for name in ['start_year', 'start_month', 'start_day']):
                    try:
                        session['start_date'] = date(c['start_year'], c['start_month'], c['start_day']).strftime('%Y%m%d')
                    except TypeError:
                        #we don't have a valid start date, skip it
                        continue
                else:
                    #missing a start date, skip it
                    continue
                course['sessions'].append(session)
            self.course_data.append(course)

        return self.course_data

    def __get_course_detail(self, id):
        response = requests.get("https://www.coursera.org/maestro/api/topic/information?topic-id=" + id)
        return response.json()
```

### Unit testing
In order to unit test I used [nose](https://nose.readthedocs.org/en/latest/) as my test runner because that is what I am most familiar with.  Here is what my unit test looks like.

``` python
import unittest
from mock import Mock, patch
from providers.coursera import Coursera
import json


#setup our mock responses
class ListResponse:
    def json(self):
        with open('test/stubs/coursera_courses.json') as f:
            data = f.read()
            courses = json.loads(data)
        return courses


class CourseResponse:
    def json(self):
        with open('test/stubs/coursera_course.json') as f:
            data = f.read()
            course = json.loads(data)
        return course


class BadCourseResponse:
    def json(self):
        with open('test/stubs/coursera_course2.json') as f:
            data = f.read()
            course = json.loads(data)
        return course


#define urls to response object
def get(*args):
    if args[0] == "https://www.coursera.org/maestro/api/topic/list?full=1":
        return ListResponse()
    elif args[0] == "https://www.coursera.org/maestro/api/topic/information?topic-id=ml":
        return CourseResponse()
    elif args[0] == "https://www.coursera.org/maestro/api/topic/information?topic-id=rt":
        return BadCourseResponse()
    else:
        print "nothing"


#test coursera provider
class CourseraTest(unittest.TestCase):
    def setUp(self):
        pass

    @patch('requests.get')
    def test_input(self, MockClass):
        MockClass.side_effect = get

        coursera = Coursera()
        courses = getattr(coursera, "get_courses")()
        self.assertTrue(MockClass.called)
        self.assertEquals(len(courses), 3)
        self.assertEquals(courses[0].get("course_name"), "Machine Learning")
        self.assertEquals(courses[2].get("language"), "japanese")

```

Using nose I can call this test with the ```nosttests``` command.  However, I like to check my code coverage as well so I installed the coverage package and I can test my providers like this:

``` bash
(wisdom)[brett:~/src/wisdom (master)]$ nosetests providers.coursera --with-coverage

Name                 Stmts   Miss  Cover   Missing
--------------------------------------------------
providers                1      0   100%
providers.coursera      55      1    98%   90
providers.provider      14      1    93%   11
--------------------------------------------------
TOTAL                   70      2    97%
----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK
```

In my [next post](/post/full_stack_mooc_app_part_1_continued_python_backend/) I will show my storage implementations. Initially I decided to build an engine for mongodb and postgres. However I used sqlalchemy so it will support any RDBMS that SQL alchemy supports.  You can take an early peek at the github page linked below.  There you will find the final version of the wisdom backend. I used virtualenv and a yaml config file for the final version so it is a little different than this post.


**[Part 1 continued, mongodb storage engine](/post/full_stack_mooc_app_part_1_continued_python_backend/)**


#### Project Wisdom Code:

**<a class="btn btn-large" href="http://www.github.com/brettdanger"><i class=" icon-github" target="_blank"></i>&nbsp;&nbsp;Wisdom Backend</a>**