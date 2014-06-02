---
layout: post
title:  "Playing with REALTIME data, Python and D3"
date:   2014-05-01 10:34:22
categories: programming
popular: true
tags: [twitter, realtime, d3.js, python, tweety, flask, rabbitmq]
description: Playing with realtime data using the twitter stream, d3.js and RabbitMQ
teaser: "/static/img/tag_cloud.png"
excerpt: I wanted a reason to play with RabbitMQ and D3 for awhile. So this seemed like a good way to do both.  I animated a tag cloud that is filled with words from a single twitter topic fed in realtime.
---
I am taking a break from my [MOOC Aggregator project](/post/full_stack_mooc_application_intro/) to play with D3.js and some realtime data. I recently read this great blog post about [performing some data analysis with python using the twitter stream](http://www.danielforsyth.me/analyzing-a-nhl-playoff-game-with-twitter/) and I wanted to take it another step and play with the data in realtime. So instead of dumping the data into mongodb I wanted to fill a queue in order to play with the data as it flows in.  I decided to use [RabbitMQ](https://www.rabbitmq.com/) as a AMQP provider.

RabbitMQ is simple to install on a mac: ```brew install rabbitmq```. This installs the queue and some more common plugins.  *See the [RabbitMQ](https://www.rabbitmq.com/download.html) installation page for help installing on other systems*.  Once it was installed I started it up with the ```rabbitmq-server``` command.

<!--more-->
Now I am ready to feed data into my queue. For this I used [Daniel Forsyth's](http://www.danielforsyth.me/analyzing-a-nhl-playoff-game-with-twitter/) example that grabs a topic from the twitter stream and stores it into mongodb.I replaced the mongo code with activemq code. I used [Pika](http://pika.readthedocs.org/en/latest/) for the python AMQP client. Here is my modified feed consumer:

```python
import tweepy
import sys
import pika
import json
import time

#get your own twitter credentials at dev.twitter.com
consumer_key = ""
consumer_secret = ""
access_token = ""
access_token_secret = ""

auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth)

class CustomStreamListener(tweepy.StreamListener):
    def __init__(self, api):
        self.api = api
        super(tweepy.StreamListener, self).__init__()

        #setup rabbitMQ Connection
        connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        self.channel = connection.channel()

        #set max queue size
        args = {"x-max-length": 2000}

        self.channel.queue_declare(queue='twitter_topic_feed', arguments=args)

    def on_status(self, status):
        print status.text, "\n"

        data = {}
        data['text'] = status.text
        data['created_at'] = time.mktime(status.created_at.timetuple())
        data['geo'] = status.geo
        data['source'] = status.source

        #queue the tweet
        self.channel.basic_publish(exchange='',
                                    routing_key='twitter_topic_feed',
                                    body=json.dumps(data))

    def on_error(self, status_code):
        print >> sys.stderr, 'Encountered error with status code:', status_code
        return True  # Don't kill the stream

    def on_timeout(self):
        print >> sys.stderr, 'Timeout...'
        return True  # Don't kill the stream

sapi = tweepy.streaming.Stream(auth, CustomStreamListener(api))
# my keyword today is chelsea as the team just had a big win
sapi.filter(track=['chelsea'])  
```
*Note: I set the queue size to 2000 tweets*. 

After running this script you start to see the tweets streaming in and filling the queue:

```
(twitter_feed_cloud)[brett:~/src/twitter_feed_cloud]$ python feed_producer.py
chelsea better not park the bus again
Kick Off babak pertama, Chelsea vs Atletico Madrid #UCL
Chelsea / atletico madrid #CestPartie
Pegang Chelsea Aja dah
Chelsea vs Atletico! Kick off..
```

**Here is the rabbitMQ screenshot:**
<img src="/static/img/rabbit_mq.png" style="display: block;  width: 100%; margin: 0 auto;">

##Consuming the queue
Now that I have realtime data flowing into a RabbitMQ queue I can start to play with the data.  At this point I wasn't quite sure what I wanted to do with the data other than play with some d3 charts.  The first thing I needed to do was stand up an API that will send the data to the frontend via AJAX (at least initially).  

One thing I knew I wanted to do was aggregate and count words in the tweets.  So I used [Flask](http://flask.pocoo.org/) to standup a quick API that consumes the queue, formats the data, and sends JSON payloads to the frontend.  Consuming the queue is pretty easy with Pika: 

```python 
import pika

 #setup queue
connection = pika.BlockingConnection()
channel = connection.channel()

#function to get X messages from the queue
def get_tweets(size=10):
    tweets = []
    count = 0
    for method_frame, properties, body in channel.consume('twitter_topic_feed'):

        tweets.append(json.loads(body))
        count += 1

        # Acknowledge the message
        channel.basic_ack(method_frame.delivery_tag)

        # Escape out of the loop after 10 messages
        if count == size:
            break

    # Cancel the consumer and return any pending messages
    requeued_messages = channel.cancel()
    print 'Requeued %i messages' % requeued_messages

    return tweets
```

Here is the full Flask API with a route to get the word counts from X number of tweets:

```python
from flask import Flask, Response
import pika
import json
import pandas

 #setup queue
connection = pika.BlockingConnection()
channel = connection.channel()


#function to get data from queue
def get_tweets(size=10):
    tweets = []
    # Get ten messages and break out
    count = 0
    for method_frame, properties, body in channel.consume('twitter_topic_feed'):

        tweets.append(json.loads(body))

        count += 1

        # Acknowledge the message
        channel.basic_ack(method_frame.delivery_tag)

        # Escape out of the loop after 10 messages
        if count == size:
            break

    # Cancel the consumer and return any pending messages
    requeued_messages = channel.cancel()
    print 'Requeued %i messages' % requeued_messages

    return tweets

app = Flask(__name__)

app.config.update(
    DEBUG=True,
    PROPAGATE_EXCEPTIONS=True
)


@app.route('/feed/raw_feed', methods=['GET'])
def get_raw_tweets():
    tweets = get_tweets(size=5)
    text = ""
    for tweet in tweets:
        tt = tweet.get('text', "")
        text = text + tt + "<br>"

    return text


@app.route('/feed/word_count', methods=['GET'])
def get_word_count():

    #get tweets from the queue
    tweets = get_tweets(size=30)

    #dont count these words
    ignore_words = [ "rt", "chelsea"]
    words = []
    for tweet in tweets:
        tt = tweet.get('text', "").lower()
        for word in tt.split():
            if "http" in word:
                continue
            if word not in ignore_words:
                words.append(word)

    p = pandas.Series(words)
    #get the counts per word
    freq = p.value_counts()
    #how many max words do we want to give back
    freq = freq.ix[0:300]

    response = Response(freq.to_json())
    
    response.headers.add('Access-Control-Allow-Origin', "*")
    return response

if __name__ == "__main__":
    app.run()
```

You will want to tune this to only process a number of tweets based on your polling rate and the rate of messages coming from twitter.  With the *Chelsea* topic I was getting about 25 tweets per second and polling this api every 3 seconds. I set the number of tweets per request to 30. I also set the number of words to give back to 300.

Here is the curl response:

``` bash
GET http://localhost:5000/feed/word_count --json
HTTP/1.0 200 OK
Access-Control-Allow-Origin: *
Content-Length: 96
Content-Type: text/html; charset=utf-8
Date: Wed, 30 Apr 2014 19:21:20 GMT
Server: Werkzeug/0.9.4 Python/2.7.6

{"de":8,"la":6,"atl\u00e9tico":6,"el":4,"madrid":4,"y":4,"vs":3,"masih":3,"champions":3,"0-0":3}
```
###Creating an animated word cloud
Now we can use this API to feed some cool frontend visualizations.  I have wanted to play with [D3](http://d3js.org/) more and I found a [word cloud D3 implementation](https://github.com/jasondavies/d3-cloud) (not that I believe word clouds are really interesting, but they look cool).  This word cloud has a useful example in the examples folder that gave me a starting point.  However, I wanted my word cloud to be bigger and to be animated with real time data. 

The first step was to create my word_cloud object:

```javascript
var fontSize = d3.scale.log().range([10, 90]);

    //create my cloud object
    var mycloud = d3.layout.cloud().size([600, 600])
          .words([])
          .padding(2)
          .rotate(function() { return ~~(Math.random() * 2) * 90; })
          .font("Impact")
          .fontSize(function(d) { return fontSize(d.size); })
          .on("end", draw)
```

Here I have a cloud object that is 600x600 pixels and it will place text that is rotated either 0 or 90 degrees.  This cloud also changes the font-size based on the size parameter passed in and it will use D3's scaling feature to scale the size to a value between 10px and 90px.  I planned to use the word count value as the size parameter.  Now we also need the draw function that actually renders the SVG graphic on screen.

Here is my draw function:

```javascript
//render the cloud with animations
     function draw(words) {

        //render new tag cloud
        d3.select("body").selectAll("svg")
            .append("g")
                 .attr("transform", "translate(300,300)")
                .selectAll("text")
                .data(words)
            .enter().append("text")
            .style("font-size", function(d) { return ((d.size)* 1) + "px"; })
            .style("font-family", "Impact")
            .style("fill", function(d, i) { return fill(i); })
            .style("opacity", 1e-6)
            .attr("text-anchor", "middle")
            .attr("transform", function(d) { return "translate(" + [d.x, d.y] + ")rotate(" + d.rotate + ")"; })
            .transition()
            .duration(1000)
            .style("opacity", 1)
            .text(function(d) { return d.text; });
```

Here I am creating a new svg group for each set of words.  The ```.append("g").attr("transform", "translate(300,300)")``` line gives the starting point for every word. The word cloud tries to place a word at this position and moves it outward if it intersects other words.  I also added a animation to fade the words in once they are all created. That animation is handled by ```.transition().duration(1000).style("opacity", 1)```.

Now I want to dynamically grab the words from my api. Here is the function that gets the words from the flask API and pushes them into my tag cloud and then renders it.

```javascript
    //ajax call
    function get_words() {
        //make ajax call
        d3.json("http://127.0.0.1:5000/feed/word_count", function(json, error) {
          if (error) return console.warn(error);
          var words_array = [];
          for (key in json){
            words_array.push({text: key, size: json[key]})
          }
          //render cloud
          mycloud.stop().words(words_array).start();
        });
    };
```

This creates this cool tag cloud:
<svg width="600" height="600"><g transform="translate(300,300)"><text text-anchor="middle" transform="translate(-21,-37)rotate(90)" style="font-size: 82px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">de</text><text text-anchor="middle" transform="translate(-143,-40)rotate(90)" style="font-size: 72px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">atlético</text><text text-anchor="middle" transform="translate(-52,17)rotate(0)" style="font-size: 72px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">la</text><text text-anchor="middle" transform="translate(-40,68)rotate(0)" style="font-size: 58px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">el</text><text text-anchor="middle" transform="translate(88,-69)rotate(90)" style="font-size: 58px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">madrid</text><text text-anchor="middle" transform="translate(-135,90)rotate(90)" style="font-size: 58px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">y</text><text text-anchor="middle" transform="translate(-70,-121)rotate(0)" style="font-size: 48px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">vs</text><text text-anchor="middle" transform="translate(-92,134)rotate(90)" style="font-size: 48px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">masih</text><text text-anchor="middle" transform="translate(43,77)rotate(90)" style="font-size: 48px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">champions</text><text text-anchor="middle" transform="translate(-74,-81)rotate(90)" style="font-size: 48px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">0-0</text><text text-anchor="middle" transform="translate(35,-132)rotate(0)" style="font-size: 48px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">del</text><text text-anchor="middle" transform="translate(32,83)rotate(0)" style="font-size: 48px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">-</text><text text-anchor="middle" transform="translate(87,136)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">madrid.</text><text text-anchor="middle" transform="translate(-30,-140)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">atletico</text><text text-anchor="middle" transform="translate(-176,-74)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">#chelsea</text><text text-anchor="middle" transform="translate(-22,98)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">0</text><text text-anchor="middle" transform="translate(-38,227)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">chelsea-atletico,</text><text text-anchor="middle" transform="translate(7,52)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">los</text><text text-anchor="middle" transform="translate(150,46)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">tutto</text><text text-anchor="middle" transform="translate(163,80)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">boring</text><text text-anchor="middle" transform="translate(139,-87)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">final</text><text text-anchor="middle" transform="translate(0,-95)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">ni</text><text text-anchor="middle" transform="translate(-151,-159)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">gioco</text><text text-anchor="middle" transform="translate(53,-176)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">come</text><text text-anchor="middle" transform="translate(-74,45)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">e</text><text text-anchor="middle" transform="translate(176,114)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">chelsea.</text><text text-anchor="middle" transform="translate(149,-161)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">saying</text><text text-anchor="middle" transform="translate(156,-28)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">en</text><text text-anchor="middle" transform="translate(52,-79)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">just</text><text text-anchor="middle" transform="translate(-174,123)rotate(90)" style="font-size: 34px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">#cheatl</text><text text-anchor="middle" transform="translate(178,8)rotate(0)" style="font-size: 34px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">costa</text><text text-anchor="middle" transform="translate(181,-66)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">gawang</text><text text-anchor="middle" transform="translate(20,18)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">got</text><text text-anchor="middle" transform="translate(-38,-41)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">31,</text><text text-anchor="middle" transform="translate(-3,69)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">tgok</text><text text-anchor="middle" transform="translate(70,-45)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">u</text><text text-anchor="middle" transform="translate(23,128)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">diego</text><text text-anchor="middle" transform="translate(-20,136)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">looking</text><text text-anchor="middle" transform="translate(-120,115)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">bosan</text><text text-anchor="middle" transform="translate(-184,5)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">@rmsoccer_27:</text><text text-anchor="middle" transform="translate(-192,-25)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">para</text><text text-anchor="middle" transform="translate(120,133)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">kene</text><text text-anchor="middle" transform="translate(65,-118)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">women.</text><text text-anchor="middle" transform="translate(-161,46)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">quiet!</text><text text-anchor="middle" transform="translate(12,78)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">dgn</text><text text-anchor="middle" transform="translate(-161,30)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">o.</text><text text-anchor="middle" transform="translate(-54,-169)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">yoruba</text><text text-anchor="middle" transform="translate(94,44)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">vamos</text><text text-anchor="middle" transform="translate(-85,-159)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">sono</text><text text-anchor="middle" transform="translate(143,151)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">@gameyetu:</text><text text-anchor="middle" transform="translate(-44,-132)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">una</text><text text-anchor="middle" transform="translate(25,94)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">táctica...</text><text text-anchor="middle" transform="translate(134,-136)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">quiet</text><text text-anchor="middle" transform="translate(16,156)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">semi-final</text><text text-anchor="middle" transform="translate(-195,-80)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">quickly</text><text text-anchor="middle" transform="translate(166,-131)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">getting</text><text text-anchor="middle" transform="translate(19,-118)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">rare</text><text text-anchor="middle" transform="translate(122,174)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">ate</text><text text-anchor="middle" transform="translate(172,129)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">league</text><text text-anchor="middle" transform="translate(-79,66)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">¿aún</text><text text-anchor="middle" transform="translate(-173,21)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">mejor</text><text text-anchor="middle" transform="translate(-86,-183)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">prefiero</text><text text-anchor="middle" transform="translate(-47,-182)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">mes</text><text text-anchor="middle" transform="translate(-216,-25)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">igualan</text><text text-anchor="middle" transform="translate(-29,-206)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">@mickf96</text><text text-anchor="middle" transform="translate(159,153)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">scores...</text><text text-anchor="middle" transform="translate(0,-205)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">numbers</text><text text-anchor="middle" transform="translate(97,29)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">cahill.</text><text text-anchor="middle" transform="translate(-108,138)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">huevos</text><text text-anchor="middle" transform="translate(-134,136)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">#atleti</text><text text-anchor="middle" transform="translate(98,57)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">foto</text><text text-anchor="middle" transform="translate(216,35)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">comienza</text><text text-anchor="middle" transform="translate(39,-68)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">se</text><text text-anchor="middle" transform="translate(17,172)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">second</text><text text-anchor="middle" transform="translate(33,-100)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">atletico)</text><text text-anchor="middle" transform="translate(-191,-124)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">league:</text><text text-anchor="middle" transform="translate(-72,-205)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">get</text><text text-anchor="middle" transform="translate(-215,-98)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">ktbfhhghhh</text><text text-anchor="middle" transform="translate(197,-117)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">amarilla</text><text text-anchor="middle" transform="translate(-192,35)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">porque</text><text text-anchor="middle" transform="translate(85,-161)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">madri</text><text text-anchor="middle" transform="translate(190,30)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">flags.</text><text text-anchor="middle" transform="translate(-19,14)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">👌</text><text text-anchor="middle" transform="translate(175,145)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">ada</text><text text-anchor="middle" transform="translate(177,-102)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">eux</text><text text-anchor="middle" transform="translate(-122,-187)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">pritili</text><text text-anchor="middle" transform="translate(6,140)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">-,-</text><text text-anchor="middle" transform="translate(-93,-72)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">,</text><text text-anchor="middle" transform="translate(-89,-202)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">koke</text><text text-anchor="middle" transform="translate(104,-204)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">#cha…</text><text text-anchor="middle" transform="translate(91,211)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">mucho</text><text text-anchor="middle" transform="translate(125,211)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">referencial</text><text text-anchor="middle" transform="translate(-5,16)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">als</text><text text-anchor="middle" transform="translate(-229,-25)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">niet</text><text text-anchor="middle" transform="translate(-230,-80)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">nuestra</text><text text-anchor="middle" transform="translate(39,-206)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">gagnait</text><text text-anchor="middle" transform="translate(8,190)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">(chelsea</text><text text-anchor="middle" transform="translate(241,71)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">bájatela...</text><text text-anchor="middle" transform="translate(203,-54)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">sekarang</text><text text-anchor="middle" transform="translate(169,186)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">gonna</text><text text-anchor="middle" transform="translate(78,-207)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">#cfc</text><text text-anchor="middle" transform="translate(35,242)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">@caroguillenespn:</text><text text-anchor="middle" transform="translate(223,-18)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">semifinale</text><text text-anchor="middle" transform="translate(-210,-150)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">league.</text><text text-anchor="middle" transform="translate(-175,60)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">iyaaa-_-</text><text text-anchor="middle" transform="translate(163,-147)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">cher</text><text text-anchor="middle" transform="translate(121,-205)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">rogol</text><text text-anchor="middle" transform="translate(196,135)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">hora</text><text text-anchor="middle" transform="translate(216,-88)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">semifinal</text><text text-anchor="middle" transform="translate(-191,-44)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">lose</text><text text-anchor="middle" transform="translate(141,205)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">harap</text><text text-anchor="middle" transform="translate(11,100)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">die</text><text text-anchor="middle" transform="translate(-48,-223)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">@diarioas:</text><text text-anchor="middle" transform="translate(208,-150)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">kedua</text><text text-anchor="middle" transform="translate(-152,-209)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">mourinho</text><text text-anchor="middle" transform="translate(-243,-35)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">centrales</text><text text-anchor="middle" transform="translate(-192,99)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">chelsea!!!!</text><text text-anchor="middle" transform="translate(86,68)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">hij</text><text text-anchor="middle" transform="translate(-76,-76)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">31</text><text text-anchor="middle" transform="translate(112,214)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">sgt</text><text text-anchor="middle" transform="translate(-211,55)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">van...</text><text text-anchor="middle" transform="translate(-121,172)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">minuto</text><text text-anchor="middle" transform="translate(182,-32)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">moi</text><text text-anchor="middle" transform="translate(27,-221)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">miedo</text><text text-anchor="middle" transform="translate(-206,76)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">veel</text><text text-anchor="middle" transform="translate(-217,20)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">pergi</text><text text-anchor="middle" transform="translate(-30,251)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">volgend</text><text text-anchor="middle" transform="translate(-52,246)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">want</text><text text-anchor="middle" transform="translate(200,163)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">tight</text><text text-anchor="middle" transform="translate(-225,100)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">gentlemen</text><text text-anchor="middle" transform="translate(-145,192)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">though.</text><text text-anchor="middle" transform="translate(-245,8)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">media</text><text text-anchor="middle" transform="translate(70,-223)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">diretta</text><text text-anchor="middle" transform="translate(-83,245)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">belum</text><text text-anchor="middle" transform="translate(104,71)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">nk</text><text text-anchor="middle" transform="translate(12,258)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">nontonnya</text><text text-anchor="middle" transform="translate(47,-241)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">tak</text><text text-anchor="middle" transform="translate(70,-146)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">mins</text><text text-anchor="middle" transform="translate(-196,144)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">alright,</text><text text-anchor="middle" transform="translate(135,-220)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">@koppnts:</text><text text-anchor="middle" transform="translate(-118,-241)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">@tweetchelseafc:</text><text text-anchor="middle" transform="translate(-239,44)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">assistir</text><text text-anchor="middle" transform="translate(-45,-151)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">al</text><text text-anchor="middle" transform="translate(-90,10)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">zo</text><text text-anchor="middle" transform="translate(-17,-237)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">partido,</text><text text-anchor="middle" transform="translate(-215,130)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">predict</text><text text-anchor="middle" transform="translate(-194,193)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">encuentran</text><text text-anchor="middle" transform="translate(30,-236)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">ae</text><text text-anchor="middle" transform="translate(240,28)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">gone</text><text text-anchor="middle" transform="translate(-11,83)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">final.</text><text text-anchor="middle" transform="translate(244,47)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">della</text><text text-anchor="middle" transform="translate(55,-254)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">#atleticodemadrid</text><text text-anchor="middle" transform="translate(232,-63)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">pura</text><text text-anchor="middle" transform="translate(-232,128)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">boys</text><text text-anchor="middle" transform="translate(109,-227)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">iyaaa,</text><text text-anchor="middle" transform="translate(-130,241)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">ptnnnnnnn</text><text text-anchor="middle" transform="translate(195,187)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">verder</text><text text-anchor="middle" transform="translate(-87,-220)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">er</text><text text-anchor="middle" transform="translate(-70,266)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">@deqtanariss:</text><text text-anchor="middle" transform="translate(-247,-94)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">primera</text><text text-anchor="middle" transform="translate(221,-104)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">dicho</text><text text-anchor="middle" transform="translate(243,134)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">masing-masing.</text><text text-anchor="middle" transform="translate(-103,-239)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">debilidad</text><text text-anchor="middle" transform="translate(155,208)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">politici…</text><text text-anchor="middle" transform="translate(-78,-239)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">twats</text><text text-anchor="middle" transform="translate(142,249)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">@koertwesterman:</text><text text-anchor="middle" transform="translate(-174,205)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">iya,</text><text text-anchor="middle" transform="translate(243,5)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">mag</text><text text-anchor="middle" transform="translate(194,-139)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">fans</text><text text-anchor="middle" transform="translate(-47,-264)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">"@yuyunnurul69:</text><text text-anchor="middle" transform="translate(-30,-75)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">a.</text><text text-anchor="middle" transform="translate(-265,-14)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">hazard</text><text text-anchor="middle" transform="translate(8,-239)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">#ucl</text><text text-anchor="middle" transform="translate(56,-274)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">@santymod</text><text text-anchor="middle" transform="translate(182,223)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">bakt</text><text text-anchor="middle" transform="translate(-6,120)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">ball</text><text text-anchor="middle" transform="translate(-257,-56)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">siso</text><text text-anchor="middle" transform="translate(-196,-15)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">x</text><text text-anchor="middle" transform="translate(-256,23)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">hoy</text><text text-anchor="middle" transform="translate(56,265)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">adrián.</text><text text-anchor="middle" transform="translate(-178,-195)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">segui...</text><text text-anchor="middle" transform="translate(-236,187)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">@jarangmandi_:</text><text text-anchor="middle" transform="translate(224,165)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">fue</text><text text-anchor="middle" transform="translate(-247,76)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">menjaga</text><text text-anchor="middle" transform="translate(-106,173)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">de...</text><text text-anchor="middle" transform="translate(-130,262)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">contre</text><text text-anchor="middle" transform="translate(168,-198)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">live</text><text text-anchor="middle" transform="translate(-33,-243)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">tim</text><text text-anchor="middle" transform="translate(-193,228)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">#realmadrid</text><text text-anchor="middle" transform="translate(-134,-217)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">feh</text><text text-anchor="middle" transform="translate(243,-32)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">conoces</text><text text-anchor="middle" transform="translate(118,-254)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">jaar</text><text text-anchor="middle" transform="translate(-266,52)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">gaat</text><text text-anchor="middle" transform="translate(-180,-208)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">peluang</text><text text-anchor="middle" transform="translate(238,88)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">ayo</text><text text-anchor="middle" transform="translate(-250,-129)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">@alfianlutfi09:</text><text text-anchor="middle" transform="translate(7,275)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">aflojan</text><text text-anchor="middle" transform="translate(-267,-84)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">partido</text><text text-anchor="middle" transform="translate(78,270)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">chelsea!</text><text text-anchor="middle" transform="translate(-228,-152)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">jogo</text><text text-anchor="middle" transform="translate(-216,-172)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">trabajo</text><text text-anchor="middle" transform="translate(-274,7)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">honest</text><text text-anchor="middle" transform="translate(211,132)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">one</text><text text-anchor="middle" transform="translate(-54,-249)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">por</text><text text-anchor="middle" transform="translate(258,-20)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">app</text><text text-anchor="middle" transform="translate(-265,94)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">smpai</text><text text-anchor="middle" transform="translate(-48,-36)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">!!</text><text text-anchor="middle" transform="translate(210,-20)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">con</text><text text-anchor="middle" transform="translate(-194,175)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">keep</text><text text-anchor="middle" transform="translate(37,-28)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">:</text><text text-anchor="middle" transform="translate(267,6)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">parei</text><text text-anchor="middle" transform="translate(-218,-58)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">da</text><text text-anchor="middle" transform="translate(30,142)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">stu</text><text text-anchor="middle" transform="translate(-132,-241)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">leg:</text><text text-anchor="middle" transform="translate(40,281)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">deu</text><text text-anchor="middle" transform="translate(-222,37)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">mi</text><text text-anchor="middle" transform="translate(-205,213)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(188, 189, 34); opacity: 1;">👏👏👏👌</text><text text-anchor="middle" transform="translate(-254,-164)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(219, 219, 141); opacity: 1;">degoute</text><text text-anchor="middle" transform="translate(8,-276)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(23, 190, 207); opacity: 1;">atm..</text><text text-anchor="middle" transform="translate(77,-237)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(158, 218, 229); opacity: 1;">atleti</text><text text-anchor="middle" transform="translate(-250,121)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(31, 119, 180); opacity: 1;">ben</text><text text-anchor="middle" transform="translate(-152,-248)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(174, 199, 232); opacity: 1;">bek</text><text text-anchor="middle" transform="translate(-212,-220)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 127, 14); opacity: 1;">vergeten</text><text text-anchor="middle" transform="translate(224,-138)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 187, 120); opacity: 1;">chato</text><text text-anchor="middle" transform="translate(-232,-193)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(44, 160, 44); opacity: 1;">badly.</text><text text-anchor="middle" transform="translate(-81,279)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(152, 223, 138); opacity: 1;">striker</text><text text-anchor="middle" transform="translate(265,-74)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(214, 39, 40); opacity: 1;">@chydee:</text><text text-anchor="middle" transform="translate(-189,-245)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(255, 152, 150); opacity: 1;">@mirondo:</text><text text-anchor="middle" transform="translate(-243,-211)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(148, 103, 189); opacity: 1;">jsupporte</text><text text-anchor="middle" transform="translate(-146,-275)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(197, 176, 213); opacity: 1;">segui</text><text text-anchor="middle" transform="translate(258,31)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(140, 86, 75); opacity: 1;">otro.</text><text text-anchor="middle" transform="translate(243,-111)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(196, 156, 148); opacity: 1;">back</text><text text-anchor="middle" transform="translate(223,-178)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(227, 119, 194); opacity: 1;">liverpool</text><text text-anchor="middle" transform="translate(-279,-52)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(247, 182, 210); opacity: 1;">móvil?</text><text text-anchor="middle" transform="translate(266,-43)rotate(0)" style="font-size: 10px; font-family: Impact; fill: rgb(127, 127, 127); opacity: 1;">vous</text><text text-anchor="middle" transform="translate(-159,270)rotate(90)" style="font-size: 10px; font-family: Impact; fill: rgb(199, 199, 199); opacity: 1;">kegadisan</text></g></svg>

Now I want to poll for new data every X seconds based on how fast data is coming in from my feed.  Here I have it set to a four second interval. ```var interval = setInterval(function(){get_words()}, 4000);``` 

Here is the animated version:
<iframe width="640" height="480" src="//www.youtube.com/embed/zE4Usr5mstc?rel=0" frameborder="0" allowfullscreen></iframe>

My final javascript word count implementation:

```javascript
<!DOCTYPE html>
<meta charset="utf-8">
<body>
<script src="d3-cloud/lib/d3/d3.js"></script>
<script src="d3-cloud/d3.layout.cloud.js"></script>
<script>
(function() {
    var fill = d3.scale.category20();
    //what range of font sizes do we want, we will scale the word counts
    var fontSize = d3.scale.log().range([10, 90]);

    //create my cloud object
    var mycloud = d3.layout.cloud().size([600, 600])
          .words([])
          .padding(2)
          .rotate(function() { return ~~(Math.random() * 2) * 90; })
          // .rotate(function() { return 0; })
          .font("Impact")
          .fontSize(function(d) { return fontSize(d.size); })
          .on("end", draw)

    //render the cloud with animations
     function draw(words) {
        //fade existing tag cloud out
        d3.select("body").selectAll("svg").selectAll("g")
            .transition()
                .duration(1000)
                .style("opacity", 1e-6)
                .remove();

        //render new tag cloud
        d3.select("body").selectAll("svg")
            .append("g")
                 .attr("transform", "translate(300,300)")
                .selectAll("text")
                .data(words)
            .enter().append("text")
            .style("font-size", function(d) { return ((d.size)* 1) + "px"; })
            .style("font-family", "Impact")
            .style("fill", function(d, i) { return fill(i); })
            .style("opacity", 1e-6)
            .attr("text-anchor", "middle")
            .attr("transform", function(d) { return "translate(" + [d.x, d.y] + ")rotate(" + d.rotate + ")"; })
            .transition()
            .duration(1000)
            .style("opacity", 1)
            .text(function(d) { return d.text; });
      }

    //ajax call
    function get_words() {
        //make ajax call
        d3.json("http://127.0.0.1:5000/feed/word_count", function(json, error) {
          if (error) return console.warn(error);
          var words_array = [];
          for (key in json){
            words_array.push({text: key, size: json[key]})
          }
          //render cloud
          mycloud.stop().words(words_array).start();
        });
    };

    //create SVG container
    d3.select("body").append("svg")
        .attr("width", 600)
        .attr("height", 600);

    //render first cloud
    get_words();

    //start streaming
    //var interval = setInterval(function(){get_words()}, 4000);
})();
</script>
```

###What's Next?

- Use Node.js and socket.io to push the data to the client instead of polling it. I can use a node.js AMQP client to grab data off the queue.
- Create other cool visualizations. Maybe a realtime geo location map or a animated chart showing client usage shares.
- Any other cool ideas? Comment below.
