---
layout: post
title:  "Mooc Aggregator Project, Python backend Continued, saving to mongodb and postgres"
date:   2014-04-22 10:34:22
categories: programming
tags: [github, python, sqlalchemy, postgres, mongodb, coursera]
description: "Python Backend for online course aggregator. Building a modular backend with Python that supports multiple course providers and storage engines. This is part 1 with support for coursera and a storage engine for mongodb and relational databases using SQLAlchemy."
teaser: "/static/img/mooc_python.png"
---

### Part 1 Continued - Saving to mongodb


I decided to build an online course aggregator and introduced the project in [this](/post/full_stack_mooc_application_intro/) post, and [last week](/post/full_stack_mooc_app_part_1_python_backend/) I started part 1 explaining the backend and creating the first provider class for coursera. This post I will dive into the storage engines, initially I created two storage engines; mongodb and postgres using sql alchemy. 

<!--more-->
The storage engine is similar to the provider model, where I create an abstract base class that contains the store_courses() abstract method and each engine will implement that method.  Here is the main program which iterates over a list of providers and collects courses, then iterates over the storage engines saving the courses.

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

Here we loop collect the data from the provider, then save that providers data for every enabled storage engine. Since my provider class returns a specific JSON structure creating the mongodb engine was very simple as we just need to store that json doc.  First things first, here is the basic storage engine abstract base class.

``` python

from abc import ABCMeta, abstractmethod
import yaml


class StorageBase(object):
    __metaclass__ = ABCMeta

    @classmethod
    def get_config(cls, engine_name):
        config = yaml.load(open('config.yaml'))
        return config['storage_configs'].get(engine_name, None)

    @abstractmethod
    def store_courses(self):
        #abstract implemented in each provider
        pass

```

Once again we are using python's ABC library to create the abstract class.  I also created a get_config class method to get settings from a configuration file.


### MongoDB Storage Engine

The mongodb simply needs to iterate over each course and save the specific course in a collection. However, I plan on running this program on a semi-regular basis and I only want to update existing data so I don't have duplication.  Unfortunately I could not find an easy way to perform a bulk upsert, so I am  iterating and upserting individually. This seems to work well enough.  Here is what the mongoengine looks like.

``` python
from storage import StorageBase
import pymongo
from pymongo import ASCENDING


class MongoDB(StorageBase):
    #do not override here, place this in the YAML configuration config.yaml
    config = {
        "server": "localhost",
        "db": "wisdom",
        "collection": "courses"}

    def __init__(self):
        config = self.get_config(self.__class__.__name__)  # call the super init method to get config

        if config is not None:
            for item in config:
                self.config[item] = config[item]

        self.collection = self.config["collection"]

        connection = pymongo.MongoClient(self.config["server"])
        self.db = connection[self.config["db"]]

    def store_courses(self, courses):
        #we can't bulk upsert so lets loop and upsert
        for course in courses:
            self.db[self.collection].update({"id": course["id"]}, course, upsert=True)

        self.__set_indicies()

    #this method is to override the collection for testing
    def set_collection(self, collection):
        self.collection = collection

    #setup any indexes we might want
    def __set_indicies(self):
        self.db[self.collection].ensure_index("id", unique=True)
        self.db[self.collection].ensure_index([("course_name", ASCENDING), ("provider", ASCENDING)], unique=True)
```

We have a default configuration in case the collection, server and db names aren't set in the config file.  The store_courses implementation with mongo is very easy.

``` python 
def store_courses(self, courses):
    #we can't bulk upsert so lets loop and upsert
    for course in courses:
        self.db[self.collection].update({"id": course["id"]}, course, upsert=True)

    self.__set_indicies()

```

We simply call pymongo's update method and set upsert to true so it adds docs that don't exist.  I also added a private method to create indexes on the id and course_name fields so we can have fast searches on those fields.

### Unit Testing the mongo class

Once again I use a test driven approach not only because it is a best practice, but also because I don't want to hammer the coursera API over and over while I code the storage engines. In the test folder there is a stubs folder that contains a ```collected_courses.json``` file that has a small subset of the data we get back from the providers class. Using this stub, I create a unit test for mongodb that saves this data into a test collection and makes sure the data is upserted properly.

``` python

import unittest
from storage.mongodb import MongoDB
import pymongo
import json
import yaml


class MongoTest(unittest.TestCase):
    def setUp(self):
        self.config = {
            "unit_test_collection": "testing"
        }
        #get the configuration
        configs = yaml.load(open('config.yaml'))['storage_configs']["MongoDB"]
        for item in configs:
            self.config[item] = configs[item]

        with open('test/stubs/collected_courses.json') as f:
            data = f.read()
            self.courses = json.loads(data)

        mongodb = pymongo.MongoClient()
        self.mongo = mongodb[self.config['db']]

        #drop the unit_test collection
        self.mongo[self.config['unit_test_collection']].drop()

    def test_input(self):
        mDB = MongoDB()
        #overide to testing collection
        mDB.set_collection(self.config["unit_test_collection"])
        mDB.store_courses(self.courses)

        #now check the results
        count = self.mongo[self.config["unit_test_collection"]].count()

        self.assertEquals(count, 10)

    #reinsert and make sure it still equals 10
    def test_updates(self):
        self.test_input()

    #make a change to a few records and make sure the data is changed
    def test_changes(self):
        self.courses[1]["short_description"] = "A new short description"
        self.courses[0]["providers_id"] = "unittests"

        #reenter the data
        self.test_input()

        #check for test_updates
        course = self.mongo[self.config["unit_test_collection"]].find({"id": "daf1d740782b94d750502fcb18a284fa"})
        self.assertEquals(course[0]["short_description"], "A new short description")

        #check for test_updates
        course = self.mongo[self.config["unit_test_collection"]].find({"id": "d114184968727a76272e32f6e4417e78"})
        self.assertEquals(course[0]["providers_id"], "unittests")

        #make sure we still have 10 records
        self.assertEquals(self.mongo[self.config["unit_test_collection"]].count(), 10)
```

After running the unit test I can go into mongodb and look at the data in the unit_test collection.

``` bash

[Brett@bretts-mac-mini:~/src/published_blog (master)]$ mongo
MongoDB shell version: 2.2.2
connecting to: test
> show dbs
wisdom  0.203125GB
> use wisdom
switched to db wisdom
> show collections
courses
system.indexes
unit_tests
> db.unit_tests.count()
10
> db.unit_tests.findOne()
{
    "_id" : ObjectId("533cc8ab61d30a89d1d279ae"),
    "workload" : "8-10 hours/week, 10-20 hours/week with programming assignments",
    "course_name" : "Compilers",
    "language" : "english",
    "sessions" : [
        {
            "duration" : "10 weeks",
            "provider_session_id" : 76,
            "start_date" : "20120423"
        },
        {
            "duration" : "11 weeks",
            "provider_session_id" : 150,
            "start_date" : "20121001"
        },
        {
            "duration" : "11 weeks",
            "provider_session_id" : 970514,
            "start_date" : "20130211"
        },
        {
            "duration" : "11 weeks",
            "provider_session_id" : 972209,
            "start_date" : "20140317"
        }
    ],
    "media" : {
        "icon_url" : "https://s3.amazonaws.com/coursera/topics/compilers/large-icon.png",
        "video_id" : "sm0QQO-WZlM",
        "video_url" : "https://d1a2y8pfnfh44t.cloudfront.net/sm0QQO-WZlM/",
        "video_type" : "mp4",
        "photo_url" : "https://s3.amazonaws.com/coursera/topics/compilers/large-icon.png"
    },
    "providers_id" : "compilers",
    "tags" : [
        "Computer Science: Systems & Security",
        "Computer Science: Software Engineering"
    ],
    "institution" : {
        "website" : "http://online.stanford.edu/",
        "city" : "Palo Alto",
        "name" : "Stanford University",
        "country" : "US",
        "state" : "CA",
        "logo_url" : "https://coursera-university-assets.s3.amazonaws.com/d8/4c69670e0826e42c6cd80b4a02b9a2/stanford.png",
        "id" : "269fc60adb004b0b719031a97aedf5e9",
        "description" : "The Leland Stanford Junior University, commonly referred to as Stanford University or Stanford, is an American private research university located in Stanford, California on an 8,180-acre (3,310 ha) campus near Palo Alto, California, United States."
    },
    "full_description" : "<p>This course will discuss the major ideas used today in the implementation of programming language compilers, including lexical analysis, parsing, syntax-directed translation, abstract syntax trees, types and type checking, intermediate languages, dataflow analysis, program optimization, code generation, and runtime systems. As a result, you will learn how a program written in a high-level language designed for humans is systematically translated into a program written in low-level assembly more suited to machines. Along the way we will also touch on how programming languages are designed, programming language semantics, and why there are so many different kinds of programming languages.</p>\n<p>The course lectures will be presented in short videos. To help you master the material, there will be in-lecture questions to answer, quizzes, and two exams: a midterm and a final. There will also be homework in the form of exercises that ask you to show a sequence of logical steps needed to derive a specific result, such as the sequence of steps a type checker would perform to type check a piece of code, or the sequence of steps a parser would perform to parse an input string. This checking technology is the result of ongoing research at Stanford into developing innovative tools for education, and we're excited to be the first course ever to make it available to students.</p>\nAn optional course project is to write a complete compiler for COOL, the Classroom Object Oriented Language. COOL has the essential features of a realistic programming language, but is small and simple enough that it can be implemented in a few thousand lines of code. Students who choose to do the project can implement it in either C++ or Java.<br>I hope you enjoy the course!\n<p><strong>Why Study Compilers?</strong></p>\nEverything that computers do is the result of some program, and all of the millions of programs in the world are written in one of the many thousands of programming languages that have been developed over the last 60 years. Designing and implementing a programming language turns out to be difficult; some of the best minds in computer science have thought about the problems involved and contributed beautiful and deep results. Learning something about compilers will show you the interplay of theory and practice in computer science, especially how powerful general ideas combined with engineering insight can lead to practical solutions to very hard problems. Knowing how a compiler works will also make you a better programmer and increase your ability to learn new programming languages quickly.\n<div></div>",
    "provider" : "coursera",
    "short_description" : "This course will discuss the major ideas used today in the implementation of programming language compilers. You will learn how a program written in a high-level language designed for humans is systematically translated into a program written in low-level assembly more suited to machines!",
    "course_url" : "http://class.coursera.org/compilers/",
    "instructor" : "Alex Aiken, Professor",
    "id" : "d114184968727a76272e32f6e4417e78",
    "categories" : [
        {
            "id" : 11,
            "name" : "Computer Science: Systems & Security",
            "mailing_list_id" : null,
            "short_name" : "cs-systems",
            "description" : "Our wide range of courses allows students to explore topics from many different fields of study. Sign up for a class today and join our global community of students and scholars!"
        },
        {
            "id" : 12,
            "name" : "Computer Science: Software Engineering",
            "mailing_list_id" : null,
            "short_name" : "cs-programming",
            "description" : "Our wide range of courses allows students to explore topics from many different fields of study. Sign up for a class today and join our global community of students and scholars!"
        }
    ]
}
>
```

### Saving to a relational database

In addition to mongodb, I also wanted to save the data to a relational database as the data is naturally relational.  I choose to use the ORM SQL Alchemy as the data should be very simple to model and this will allow me the flexibility to use any major SQL implementation.  I decided to use postgres, but switching to mysql should be a simple single line change.

While mongo was a very easy implementation the SQL version is more complex as we need to setup the schema.  Fortunately SQL ALchemy makes this very simple. In the ```storage``` folder I created a ```sql_setup``` folder that has the one off scripts to run to setup the schemas. 

Here is the ```db_setup.py``` script that creates the schema.

```python
from sqlalchemy import Column, ForeignKey, Date, String, Text, Integer, Table
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy import create_engine
import yaml


def get_settings():
    config = yaml.load(open('config.yaml'))
    return config['storage_configs'].get("SQL", None)
 
Base = declarative_base()
 

course_tag_association_table = Table(
    'course_tag_association',
    Base.metadata,
    Column('course_id', String(50), ForeignKey('course.id')),
    Column('tag_id', Integer, ForeignKey('tag.id'))
)

course_category_association_table = Table(
    'course_category_association',
    Base.metadata,
    Column('course_id', String(50), ForeignKey('course.id')),
    Column('category_id', Integer, ForeignKey('category.id'))
)


class Institution(Base):
    __tablename__ = 'institution'
    # Here we define columns for the table address.
    # Notice that each column is also a normal Python instance attribute.
    id = Column(String(50), primary_key=True)
    name = Column(String(250), nullable=False)
    description = Column(Text)
    website = Column(String(250))
    logo_url = Column(String(250))
    city = Column(String(250))
    state = Column(String(250))
    country = Column(String(250))


class Course(Base):
    __tablename__ = "course"
    id = Column(String(50), primary_key=True)
    name = Column(String(250), nullable=False)
    language = Column(String(250))
    instructor = Column(String(250))
    providers_id = Column(String(50), nullable=False)
    short_description = Column(Text)
    full_description = Column(Text)
    course_url = Column(String(250))
    workload = Column(String(250))
    institution_id = Column(String(50), ForeignKey('institution.id'))
    institution = relationship(Institution)
    tags = relationship("Tag", secondary=course_tag_association_table)
    categories = relationship("Category", secondary=course_category_association_table)


class Session(Base):
    __tablename__ = "session"
    id = Column(Integer, primary_key=True)
    course_id = Column(String(50), ForeignKey('course.id'))
    course = relationship(Course)
    duration = Column(String(250))
    start_date = Column(Date)


class Media(Base):
    __tablename__ = "media"
    id = Column(Integer, primary_key=True)
    course_id = Column(String(50), ForeignKey('course.id'))
    course = relationship(Course)
    photo_url = Column(String(250))
    icon_url = Column(String(250))
    video_url = Column(String(250))
    video_type = Column(String(20))
    video_id = Column(String(50))


class Tag(Base):
    __tablename__ = "tag"
    id = Column(Integer, primary_key=True)
    tag = Column(String(50), unique=True, nullable=False)


class Category(Base):
    __tablename__ = "category"
    id = Column(Integer, primary_key=True)
    name = Column(String(250), unique=True, nullable=False)
    description = Column(Text)
 
# Create an engine that stores data in the local directory's
# sqlalchemy_example.db file.
engine = create_engine(get_settings()['connection_string'])
 
# Create all tables in the engine. This is equivalent to "Create Table"
# statements in raw SQL.
Base.metadata.create_all(engine)
```

It should be pretty simple to follow, however if you don't have any experience with ORMs and SQL Alchemy [start here](http://docs.sqlalchemy.org/en/rel_0_9/orm/tutorial.html).  To create a schema, assuming you have your sql implementation installed and running, simply edit the connection_string statement in the ```config.yaml``` file.

```yaml

#configure the wisdom system
providers: [Coursera]
storage_engines: [SQL, MongoDB]

storage_configs:
    MongoDB:
        db: wisdom
        unit_test_collection: unit_tests
    SQL:
        connection_string: postgresql://wisdom_backend:wisdom_backend@localhost/wisdom
```

Then run the db_setup.py script from the project root (since the yaml file is in the root).

Now that the schema is setup, we can create the SQL engine to save the courses.  In our storage engine we need to import the table Classes from the db_setup file like this:

```python

from sql_setup.db_setup import Institution, Base, Course, Session, Media, Tag, Category
```

We also want to inherit the StorageBase and implement the store_courses() method. The implementation is simply a matter of mapping the JSON from our providers to the fields in each table.  We also use the ```session.merge``` function to upsert the data so we don't create duplicates if we run the import weekly.

Here is the final SQL storage implementation.

```python 

from sqlalchemy.engine import create_engine
from sqlalchemy.orm import sessionmaker
from sql_setup.db_setup import Institution, Base, Course, Session, Media, Tag, Category
from datetime import datetime
from storage import StorageBase
 

class SQL(StorageBase):
    #do not override here, place this in the YAML configuration config.yaml
    
    def __init__(self):
        config = self.get_config(self.__class__.__name__)  # call the super init method to get config

        engine = create_engine(config['connection_string'])
        # Bind the engine to the metadata of the Base class so that the
        # declaratives can be accessed through a instance
        Base.metadata.bind = engine
         
        DBSession = sessionmaker(bind=engine)
        self.session = DBSession()

    def store_courses(self, courses):
        session = self.session
        #we can't bulk upsert so lets loop and upsert
        for course in courses:

            i = course['institution']
            new_institution = Institution(
                id=i.get("id"),
                name=i.get("name", "None"),
                description=i.get("name"),
                website=i.get("website"),
                logo_url=i.get("logo_url"),
                city=i.get("city"),
                state=i.get("state"),
                country=i.get("country")
            )

            new_institution = session.merge(new_institution)

            session.add(new_institution)
            session.commit()

            new_course = Course(
                id=course.get("id"),
                name=course.get("course_name"),
                language=course.get("language"),
                instructor=course.get("instructor"),
                providers_id=course.get("providers_id"),
                short_description=course.get("short_description"),
                full_description=course.get("full_description"),
                course_url=course.get("course_url"),
                workload=course.get("workload"),
                institution=new_institution
            )

            new_course = session.merge(new_course)
            session.add(new_course)
            session.commit()

            for s in course['sessions']:
                new_session = Session(
                    id=s.get("provider_session_id"),
                    duration=s.get("duration", None),
                    start_date=datetime.strptime(s.get("start_date"), "%Y%m%d"),
                    course=new_course
                )

                new_session = session.merge(new_session)
                session.add(new_session)
                session.commit()

            media = course['media']
            new_media = session.query(Media).filter_by(course=new_course).first()
            if not new_media:
                new_media = Media(
                    course=new_course,
                    photo_url=media.get("photo_url", None),
                    video_url=media.get("video_url", None),
                    video_type=media.get("video_type", None),
                    video_id=media.get("video_id", None),
                    icon_url=media.get("icon_url", None)
                )
            session.add(new_media)
            session.commit()

            for t in course['tags']:
                new_tag = session.query(Tag).filter_by(tag=t).first()
                if not new_tag:
                    new_tag = Tag(tag=t)
                new_course.tags.append(new_tag)
                session.commit()

            for c in course['categories']:
                new_cat = Category(
                    id=c.get("id"),
                    name=c.get("name"),
                    description=c.get("description", None)
                )
                new_cat = session.merge(new_cat)
                new_course.categories.append(new_cat)
                session.commit()
```

After running the import, we can look in our database and see our imported data. Here is a listing of courses at The University of Florida.

<img src="/static/img/postgres.png"/>


## Project Wisdom Code:

**<a class="btn btn-large" href="http://www.github.com/brettdanger"><i class=" icon-github" target="_blank"></i>&nbsp;&nbsp;Wisdom Backend</a>**