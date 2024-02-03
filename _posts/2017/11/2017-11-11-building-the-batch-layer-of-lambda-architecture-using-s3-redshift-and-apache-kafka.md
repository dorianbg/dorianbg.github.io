---
title: "Implementing the Batch Layer of Lambda Architecture using S3, Redshift and Apache Kafka"
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

{% include embed/youtube.html id='4G1bOOMUjw8' %} 

and you can find the source code of the series [here](https://github.com/dorianbg/lambda-architecture-demo).

* * *

### Purpose in Lambda Architecture:

- store all the tweets that were produced by Kafka Producer into S3
- export them into Redshift
- perform aggregation on the tweets to get the desired output of batch layer
- achieve this by:
    - every couple of hours get the latest unseen tweets produced by the Kafka Producer and store them into a S3 archive
    - every night run a sql query to compute the result of batch layer

### Contents:

- [Defining the Kafka consumer](#1)
- [Defining a Amazon Web Services S3 storage client](#2)
- [Writing the data into a S3 bucket](#3)
- [Exporting data from S3 bucket to Amazon Redshift using COPY command](#4)
- [Aggregating "raw" tweets in Redshift](#5)
- [Deployment](#6)

### Required libraries

```python
from kafka import KafkaConsumer
from io import StringIO
import boto3
import time
import random
```

### Defining the Kafka consumer

- setting the location of Kafka Broker
- specifying the group_id and consumer_timeout
- subsribing to a topic

```python
consumer = KafkaConsumer(
                        bootstrap_servers='localhost:9092',
                        auto_offset_reset='latest',  # Reset partition offsets upon OffsetOutOfRangeError
                        group_id='test',   # must have a unique consumer group id 
                        consumer_timeout_ms=10000)  
                                # How long to listen for messages - we do it for 10 seconds 
                                # because we poll the kafka broker only each couple of hours

consumer.subscribe('tweets-lambda1')
```

### Defining a Amazon Web Services S3 storage client

- setting the autohrizaition and bucket

```python
s3_resource = boto3.resource(
    's3',
    aws_access_key_id='x',
    aws_secret_access_key='x',
)

s3_client = s3_resource.meta.client
bucket_name = 'lambda-architecture123'
```

### Writing the data into a S3 bucket

- polling the Kafka Broker
- aggregating the latest messages into a single object in the bucket

```python
def store_twitter_data(path):
    csv_buffer = StringIO() # S3 storage is object storage -> our document is just a large string

    for message in consumer: # this acts as "get me an iterator over the latest messages I haven't seen"
        csv_buffer.write(message.value.decode() + 'n') 

    s3_resource.Object(bucket_name,path).put(Body=csv_buffer.getvalue())
```

### Exporting data from S3 bucket to Amazon Redshift using COPY command

- authenticate and create a connection using psycopg module
- export data using COPY command from S3 to Redshift "raw" table

```python
import psycopg2
config = { 'dbname': 'lambda', 
           'user':'x',
           'pwd':'x',
           'host':'data-warehouse.x.us-east-1.redshift.amazonaws.com',
           'port':'5439'
         }
conn =  psycopg2.connect(dbname=config['dbname'], host=config['host'], 
                              port=config['port'], user=config['user'], 
                              password=config['pwd'])
```

```python
def copy_files(conn, path):
    curs = conn.cursor()
    curs.execute(""" 
        copy 
        batch_raw
        from 
        's3://lambda-architecture123/""" + path + """'  
        access_key_id 'x'
        secret_access_key 'x'
        delimiter ';'
        region 'eu-central-1'
    """)
    curs.close()
    conn.commit()
```

### Computing the batch layer output

- querying the raw tweets stored in redshift to get the desired batch layer output

```python
def compute_batch_layer(conn):
    curs = conn.cursor()
    curs.execute(""" 
        drop table if exists batch_layer;

        with raw_dedup as (
        SELECT
            distinct id,created_at,followers_count,location,favorite_count,retweet_count
        FROM
            batch_raw
        ),
        batch_result as (SELECT
            location,
            count(id) as count_id,
            sum(followers_count) as sum_followers_count,
            sum(favorite_count) as sum_favorite_count,
            sum(retweet_count) as sum_retweet_count
        FROM
            raw_dedup
        group by 
            location
        )
        select 
            *
        INTO
            batch_layer
        FROM
            batch_result""")
    curs.close()
    conn.commit()
```

```
# compute_batch_layer(conn)
```

### Deployment

- perform the task every couple of hours and wait in between

```python
def periodic_work(interval):
    while True:
        path = 'apple-tweets/'+ time.strftime("%Y/%m/%d/%H") + '_tweets_' + str(random.randint(1,1000)) + '.log'
        store_twitter_data(path)
        time.sleep(interval/2)
        copy_files(conn, path)
        #interval should be an integer, the number of seconds to wait
        time.sleep(interval/2)

periodic_work(60* 4) ## 4 minutes !
```

```python
# run at the end of the day
compute_batch_layer(conn)

conn.close()
```

 

You can find the code from this blog post in [this](https://github.com/dorianb96/lambda-architecture-demo/blob/master/Implementing%20the%20batch%20layer%20of%20lambda%20architecture%20using%20S3%2C%20Redshift%20and%20Apache%20Kafka.ipynb) github repository.
