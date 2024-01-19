---
title: "Tips for easier development on AWS"
date: "2018-06-22"
categories: 
  - "data-engineering"
---

I wanted to share a couple of tips for easier development on AWS.

## 1. Use a local version of AWS for development and testing

Try something like [localstack](https://github.com/localstack/localstack) to stand up a local AWS environment. This will run AWS API compliant mock applications on your local machine.

This way you can eg. create a kinesis stream, put data into it, process it and put into local S3 without spending any resources on AWS. Furthermore, this code can be translated into production code by just replacing the client instantiations.

Localstack can be easily installed with pip:

> pip install localstack

Then you can run it with:

> localstack start

These are the services which will spin-up:

![Screen Shot 2018-06-22 at 15.19.41.png](assets/img/old_blog_post_images/screen-shot-2018-06-22-at-15-19-41.png)

In the section below we will using boto3 connect to a Kinesis stream ran by localstack.

## 2. Use pyboto3 with Python in PyCharm for auto-completion

The boto3 library is very popular for development on AWS since it's quickly adaptable to AWS API changes. The issue is that boto3 doesn't actually implement the methods specified in the API.

If you like to use Python for development on AWS, you probably want to also use and IDE like PyCharm for it's many features including auto-completion. Unfortunately auto-completion does not work for boto3.

So if you want to get auto-completion in PyCharm with boto3, I highly recommend using the [pyboto3](https://github.com/wavycloud/pyboto3) library.

It's extremely easy to install:

> pip install pyboto

Now, to use it with PyCharm, you should add a line below when defining the boto3 client like in this example with Kinesis:

![pic1](assets/img/old_blog_post_images/pic1.png)![pic2](assets/img/old_blog_post_images/pic2.png)![Screen Shot 2018-06-22 at 15.34.02.png](assets/img/old_blog_post_images/screen-shot-2018-06-22-at-15-34-02.png)
