---
title: "Apache Airflow for data pipelines and ETL management"
date: "2018-02-01"
categories: 
  - "data-engineering"
---

I have been looking for good workflow management software and found Apache Airflow to be superior to other solutions.

I've taken some time to write a pretty detailed blog post on using Airflow for development of ETL pipelines.

Airflow is a great tool which allows you to:

- centrally manage and track the execution of all your ETL jobs using a web UI
- manage shared connections to databases
- implement complex dependencies between various tasks in the form of a Directed Acyclic graph

In the blog post I cover a detailed implementation of two pipelines: one from Amazon S3 to Redshift and the other one from one table in S3 to another table using an upsert. I also show you how Airflow is used for administration of tasks and log tracking among other things.

You can read the full blog post at this [link](https://sonra.io/2018/01/01/using-apache-airflow-to-build-a-data-pipeline-on-aws/).

A small preview:

![Screen Shot 2018-02-11 at 16.15.13.png](assets/img/old_blog_post_images/screen-shot-2018-02-11-at-16-15-13.png)
