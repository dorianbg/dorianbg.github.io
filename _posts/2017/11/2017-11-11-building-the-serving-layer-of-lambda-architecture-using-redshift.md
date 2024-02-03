---
title: "Implementing the Serving Layer of Lambda Architecture using Redshift"
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

{% include embed/youtube.html id='oOyY3DxmDe4' %}

and you can find the source code of the series [here](https://github.com/dorianbg/lambda-architecture-demo).

* * *

### Purpose in Lambda Architecture:

- merge the output of speed and batch layer aggregations
- achieve this by:
    - every couple of hours run the re-computation
    - use the output of batch layer as base table
    - upsert the up-to-date values of speed layer into the base table

### Contents:

- [Creating the serving layer](#1)

### Requirements

```
import psycopg2
```

### Creating the serving layer

- authenticate and create a connection using psycopg module
- create and populate a temporary table with it's base being batch layer and upserting the speed layer
- drop the current serving layer and use the above mentioned temporary table for serving layer (no downtime)

```python
config = { 'dbname': 'lambda', 
           'user':'x',
           'pwd':'x',
           'host':'data-warehouse.x.us-east-1.redshift.amazonaws.com',
           'port':'5439'
         }
conn =  psycopg2.connect(dbname=config['dbname'], host=config['host'], 
                              port=config['port'], user=config['user'], 
                              password=config['pwd'])

curs = conn.cursor()
curs.execute(""" 
    DROP TABLE IF EXISTS serving_layer_temp; 

    SELECT 
         *
    INTO 
        serving_layer_temp
    FROM 
        batch_layer ;


    UPDATE 
        serving_layer_temp
    SET
        count_id = count_id + speed_layer."count(id)",
        sum_followers_count = sum_followers_count + speed_layer."sum(followers_count)",
        sum_favorite_count = sum_favorite_count + speed_layer."sum(favorite_count)",
        sum_retweet_count = sum_retweet_count + speed_layer."sum(retweet_count)"
    FROM
        speed_layer
    WHERE 
        serving_layer_temp.location = speed_layer.location ;


    INSERT INTO 
        serving_layer_temp
    SELECT 
        * 
    FROM 
        speed_layer
    WHERE 
        speed_layer.location 
    NOT IN (
        SELECT 
            DISTINCT location 
        FROM 
            serving_layer_temp 
    ) ;
    drop table serving_layer ;
    
    alter table serving_layer_temp
    rename to serving_layer ;        
    
""")
curs.close()
conn.commit()
conn.close()
```

 

You can find the code from this blog post in [this](https://github.com/dorianb96/lambda-architecture-demo) github repository.
