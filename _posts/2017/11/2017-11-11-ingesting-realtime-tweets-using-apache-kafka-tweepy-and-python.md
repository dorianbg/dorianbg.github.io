---
title: "Ingesting realtime tweets using Apache Kafka, Tweepy and Python"
date: "2017-11-11"
categories: 
  - "data-engineering"
---

### This post is a part of a series on Lambda Architecture consisting of:

- [Introduction to Lambda Architecture]({% link _posts/2017/11/2017-11-10-introduction-to-lambda-architecture.md %})
- [Implementing Data Ingestion using Apache Kafka, Tweepy]({% link _posts/2017/11/2017-11-11-ingesting-realtime-tweets-using-apache-kafka-tweepy-and-python.md %})
- [Implementing Batch Layer using Kafka, S3, Redshift]({% link _posts/2017/11/2017-11-11-building-the-batch-layer-of-lambda-architecture-using-s3-redshift-and-apache-kafka.md %})
- [Implementing Speed Layer using Spark Structured Streaming]({% link _posts/2017/11/2017-11-11-building-the-speed-layer-of-lambda-architecture-using-structured-spark-streaming.md %})
- [Implementing Serving Layer using Redshift]({% link _posts/2017/11/2017-11-11-building-the-serving-layer-of-lambda-architecture-using-redshift.md %})

You can also follow a walk-through of the code in this Youtube video:

```liquid
{% include embed/youtube.html id='i8XIPPcOLMg' %}
```

* * *

### Purpose in Lambda Architecture:

- main data source for the lambda architecture pipeline
- uses twitter streaming API to simulate new events coming in every minute
- Kafka Producer sends the tweets as records to the Kafka Broker

### Contents:

- [Twitter setup](#1)
- [Defining the Kafka producer](#2)
- [Producing and sending records to the Kafka Broker](#3)
- [Deployment](#4)

### Required libraries

```python
import tweepy
import time
from kafka import KafkaConsumer, KafkaProducer
```

### Twitter setup

- getting the API object using authorization information
- you can find more details on how to get the authorization here: https://developer.twitter.com/en/docs/basics/authentication/overview

```python
# twitter setup
consumer_key = "x"
consumer_secret = "x"
access_token = "x"
access_token_secret = "x"
# Creating the authentication object
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
# Setting your access token and secret
auth.set_access_token(access_token, access_token_secret)
# Creating the API object by passing in auth information
api = tweepy.API(auth) 
```

A helper function to normalize the time a tweet was created with the time of our system

```python
from datetime import datetime, timedelta

def normalize_timestamp(time):
    mytime = datetime.strptime(time, "%Y-%m-%d %H:%M:%S")
    mytime += timedelta(hours=1)   # the tweets are timestamped in GMT timezone, while I am in +1 timezone
    return (mytime.strftime("%Y-%m-%d %H:%M:%S")) 
```

### Defining the Kafka producer

- specify the Kafka Broker
- specify the topic name
- optional: specify partitioning strategy

```python
producer = KafkaProducer(bootstrap_servers='localhost:9092')
topic_name = 'tweets-lambda1'
```

### Producing and sending records to the Kafka Broker

- querying the Twitter API Object
- extracting relevant information from the response
- formatting and sending the data to proper topic on the Kafka Broker
- resulting tweets have following attributes:
    - id
    - created_at
    - followers_count
    - location
    - favorite_count
    - retweet_count

```python
def get_twitter_data():
    res = api.search("Apple OR iphone OR iPhone")
    for i in res:
        record = ''
        record += str(i.user.id_str)
        record += ';'
        record += str(normalize_timestamp(str(i.created_at)))
        record += ';'
        record += str(i.user.followers_count)
        record += ';'
        record += str(i.user.location)
        record += ';'
        record += str(i.favorite_count)
        record += ';'
        record += str(i.retweet_count)
        record += ';'
        producer.send(topic_name, str.encode(record))
```

```
get_twitter_data()
```

### Deployment

- perform the task every couple of minutes and wait in between

```python
def periodic_work(interval):
    while True:
        get_twitter_data()
        #interval should be an integer, the number of seconds to wait
        time.sleep(interval)
```

```
periodic_work(60*1)  # get data every couple of minutes
```

 

You can find the code from this blog post in [this](https://github.com/dorianb96/lambda-architecture-demo) github repository.
