---
title: "Detailed plans for my GSoC project"
date: "2017-05-31"
categories: 
  - "google-summer-of-code"
---

As I write a weekly blog describing my weekly Google summer of code progress, I realized that I haven't yet presented the high level idea behind the project. So I will paste below some excerpts from the original project proposal I submitted for the project.

### Project name

Distributed EEG data storage and processing for electronic assistive system using Apache Hadoop and Spark

### Abstract

Currently the human assistive system collects the EEG data, processes it and trains customized classifiers. With the increasing number of tested subjects, the goal is to store the rapidly growing data in a distributed storage system such as Apache Hadoop. Data processing would also be implemented on the distributed system using the MapReduce framework and its higher level extension Apache Spark.

The goal of this project is to create such a scalable system which would enable storage of very large datasets, fast analysis and predictions on the data and a pleasant user interface for users to interact with the data, signal processing methods and machine learning techniques.

This system would be well documented, easy to deploy and could be used by other researcher interested in applying big data technologies for the analysis of EEG data.

### Deliverables

- Install and configure a new Hadoop cluster with supporting applications

- Establish the appropriate data importing process and a data warehouse architecture

- Adapt the current application to support distributed processing

- Provide suitable MapReduce(Spark) functions for processing and analysis of the data

- Create an application which would take as inputs location of EEG files, signal process-

ing methods, feature extraction methods, machine learning models and machine learn-

ing model evaluators and output the results in a very nice format

- The application would enable the user to load classifiers and provide reports on the

performance of a classifier

- Create good documentation for the whole project

### Plan description (in the project proposal)

Firstly, I would build the Hadoop ecosystem infrastructure. I propose to use a solution such as Cloudera Quickstart in a Docker container or Apache Ambari, a tool for deploy- ing Hadoop cluster. It is important that the solution has Apache Hue for data visualization and workflow management. It is also important to create good documentiton for the users on how to use the system.

Then we would have to plan how the data will be imported into the Hadoop distributed file system (HDFS) and in what structure it would be stored.

Next step is to adapt the current application from Guess the Number project to support big data technologies. For big data processing I propose to use Apache Spark as it has more capabilites compared to the MapReduce framework. We would have to re-write most of the current code so that it can be executed using the Apache Spark framework, but we could re-use the logic from the current implementation such as signal processing and feature extraction methods. The new implementation would apply many different signal processing methods on the raw data, perform feature extraction of the processed data and lastly do classification on the raw input data.

The reasons for choosing those tools are following: the Spark SQL library provides easy access to raw data from Hadoop and with User-defined functions we can define the sig- nal processing methods. Spark MLLib has many feature extraction methods already im- plemented and supports the notion of a data pipeline. Lastly, many classifiers already im- plemented in Spark MLLib could be used for this project.

The users of the application should be able to choose which of the many pre-defined signal processing, feature extraction and classification methods they want to use. The users preferences would be passed in as program arguments and using Apache Hue the user would have a GUI as the input method. The application would handle the logic of building the data pipelines from those arguments.

Additionally I would add the possibility to load and save classifiers in external files.

In the end, the application would have a well documented codebase, a modular structure which allows the users to specify the data pipelines and a deployable JAR file which could then be shared and re-used by other members of the community.
