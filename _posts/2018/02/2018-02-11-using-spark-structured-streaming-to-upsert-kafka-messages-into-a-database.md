---
title: "Using Spark Structured Streaming to upsert Kafka messages into a database"
date: "2018-02-11"
categories: 
  - "data-engineering"
---

I wrote a detailed and technical blog post demonstrating an integration of Spark Structured Streaming with Apache Kafka messages and Snowflake.

An overview of the content is:

- querying Twitter API for realtime tweets
- setting up a Kafka server
- producing messages with Kafka
- consuming and parsing Kafka messages with Spark Structured Streaming
- explanation of the streaming model of Spark Structured Streaming
- upserting latest data to Snowflake

You can find the full blog post [here](https://sonra.io/2017/11/20/streaming-tweets-to-snowflake-data-warehouse-with-spark-structured-streaming-and-kafka/).

A small preview:

![Screen Shot 2018-02-11 at 17.23.07.png](assets/img/old_blog_post_images/screen-shot-2018-02-11-at-17-23-07.png)
