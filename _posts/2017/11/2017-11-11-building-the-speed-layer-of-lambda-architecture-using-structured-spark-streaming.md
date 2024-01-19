---
title: "Implementing the Speed Layer of Lambda Architecture using Spark Structured Streaming"
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

{% include embed/youtube.html id='X938VfXVd9M' %}

You can find the code from this blog post in [this](https://github.com/dorianb96/lambda-architecture-demo/blob/master/Implementing%20the%20speed%20layer%20of%20lambda%20architecture%20using%20Structured%20Spark%20Streaming) github repository.

As this is a Zeppelin notebook you likely won't be able to view it on github.

* * *

### Purpose in Lambda Architecture:

- provide analytics on real time data ("intra day") which batch layer cannot efficiently achieve
- achieve this by:
    - ingest latest tweets from Kafka Producer and analtze only those for the current day
    - perform aggregations over the data to get the desired output of speed layer

### Contents:

- Configuring spark
- Spark Structured Streaming
    - Input stage - defining the data source
    - Result stage - performing transformations on the stream
    - Output stage
- Connecting to redshift cluster
- Exporting data to Redshift

### Requirements

```none
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming.ProcessingTime
import java.util.concurrent._
```

### Configuring Spark

- properly configuring spark for our workload
- defining case class for tweets which will be used later on

```python
Thread.sleep(5000)

val spark = SparkSession
  .builder()
  .config("spark.sql.shuffle.partitions","2")  // we are running this on my laptop
  .appName("Spark Structured Streaming example")
  .getOrCreate()
  
case class tweet (id: String, created_at : String, followers_count: String, location : String, favorite_count : String, retweet_count : String)

Thread.sleep(5000)
```

### Input stage - defining the data source

- using Kafka as data source we specify:
    - location of kafka broker
    - relevant kafka topic
    - how to treat starting offsets

```scala
var data_stream = spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "tweets-lambda1")
  .option("startingOffsets","earliest")  //or latest
  .load()
 
// note how similar API is to the batch version
```

### Result stage - performing transformations on the stream

- extract the value column of kafka message
- parse each row into a member of tweet class
- filter to only look at todays tweets as results
- perform aggregations

```scala
var data_stream_cleaned = data_stream
    .selectExpr("CAST(value AS STRING) as string_value")
    .as[String]
    .map(x => (x.split(";")))
    .map(x => tweet(x(0), x(1), x(2),  x(3), x(4), x(5)))
    .selectExpr( "cast(id as long) id", "CAST(created_at as timestamp) created_at",  "cast(followers_count as int) followers_count", "location", "cast(favorite_count as int) favorite_count", "cast(retweet_count as int) retweet_count")
    .toDF()
    .filter(col("created_at").gt(current_date()))   // kafka will retain data for last 24 hours, this is needed because we are using complete mode as output
    .groupBy("location")
    .agg(count("id"), sum("followers_count"), sum("favorite_count"),   sum("retweet_count"))  
```

### Output stage

- specify the following:
    - data sink - exporting to memory (table can be accessed similar to registerTempTable()/ createOrReplaceTempView() function )
    - trigger - time between running the pipeline (ie. when to do: polling for new data, data transformation)
    - output mode - complete, append or update - since in Result stage we use aggregates, we can only use Complete or Update out put mode

```scala
val query = data_stream_cleaned.writeStream
    .format("memory")
    .queryName("demo")
    .trigger(ProcessingTime("60 seconds"))   // means that that spark will look for new data only every minute
    .outputMode("complete") // could also be complete or update
    .start()
```

### Connecting to redshift cluster

- defining JDBC connection to connect to redshift

```python
//create properties object
Class.forName("com.amazon.redshift.jdbc42.Driver")

val prop = new java.util.Properties
prop.setProperty("driver", "com.amazon.redshift.jdbc42.Driver")
prop.setProperty("user", "x")
prop.setProperty("password", "x") 

//jdbc mysql url - destination database is named "data"
val url = "jdbc:redshift://data-warehouse.x.us-east-1.redshift.amazonaws.com:5439/lambda"

//destination database table 
val table = "speed_layer"
```

### Exporting data to Redshift

- "overwriting" the table with results of query stored in memory as result of the speed layer
- scheduling the function to run every hour

```python
def exportToRedshift(){
    val df = spark.sql("select * from demo")

    //write data from spark dataframe to database
    df.write.mode("overwrite").jdbc(url, table, prop)
}


val ex = new ScheduledThreadPoolExecutor(1)
val task = new Runnable { 
  def run() = exportToRedshift()
}
val f = ex.scheduleAtFixedRate(task, 1, 1, TimeUnit.HOURS)
f.cancel(false)  
```
