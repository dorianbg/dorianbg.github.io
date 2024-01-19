---
title: "Google Summer of Code 2017 Summary"
date: "2017-08-26"
categories: 
  - "google-summer-of-code"
---

This is a summary of my Google Summer of Code project containing the definition of the initial task, the implemented solution and references to show the work I have done.

### **GOAL**  

In this project I aimed to implement an application that will enable:

1. Storage of large amounts of EEG data on a remote cluster using the Hadoop distirbuted file system
2. Fast distributed processing, analysis and machine learning on the EEG data using Apache Spark and dl4j
3. User interface for managing the data and building data analysis workflows allowing the researchers to choose from different signal processing methods and machine learning techniques

### **WORK REPORT**

In order to achieve this goal, I implemented a solution with three components:

- Client GUI which is a portable Desktop Java application allowing the user to browse/manage the data on remote Hadoop Distributed File System as well as visually build data analysis workflows using feature extraction and classification methods
- Data Analysis Package which is a Java application made using Apache Spark, Apache Hadoop and Deep Learning for Java (dl4j) frameworks whose purpose is to provide a modular way of specifying data pipelines consisting of input sources, data processing, feature extraction and classification methods
- Remote Server is a Spring Boot server and the main communication point for the Client GUI with the Data Analysis Package on the Hadoop server. It listens for requests such as job submittals, fetching the results of a job, listing of trained classifiers and etc.

 

This is a demo of the Client GUI:

[youtube=https://www.youtube.com/watch?v=48r53zLVOLM&w=320&h=266]

You can find a more detailed explanation of the solution architecture [here](https://github.com/dorianb96/gsoc-architecture-explained).

### **LINKS TO GITHUB REPOSITORIES**  

Each of these components has a separate Github repository to which I am the main (and only) contributor:

- Client GUI - [https://github.com/dorianb96/EEG_ClientGUI](https://github.com/dorianb96/EEG_ClientGUI)
- Data Analysis Package - [https://github.com/dorianb96/EEG_DataAnalysisPackage](https://github.com/dorianb96/EEG_DataAnalysisPackage)
- Remote server - [https://github.com/dorianb96/EEG_RemoteServer](https://github.com/dorianb96/EEG_RemoteServer)

Although most of my work was in the above mentioned repositories, there are still a few repositories I made containing guides or documentation for the GSoC project:

- Guide on how to install Cloudera Quickstart - [https://github.com/dorianb96/Cloudera_Quickstart_Install](https://github.com/dorianb96/Cloudera_Quickstart_Install)
- Guide on how to install, configure and administer Kerberos on Cloudera Manager as well as how to setup a client connection using Kerberos - [https://github.com/dorianb96/kerberos-on-cloudera-quickstart](https://github.com/dorianb96/kerberos-on-cloudera-quickstart)
- Deployable JAR files and documentation explaining the architecture of how the three applications interact together - [https://github.com/dorianb96/gsoc-architecture-explained](https://github.com/dorianb96/gsoc-architecture-explained)

Lastly, the list of all commits I made (70+) related to this project can be found in this list:

- [https://github.com/dorianb96/EEG_RemoteServer/commits/master](https://github.com/dorianb96/EEG_RemoteServer/commits/master)
- [https://github.com/dorianb96/EEG_DataAnalysisPackage/commits/master](https://github.com/dorianb96/EEG_DataAnalysisPackage/commits/master)
- [https://github.com/dorianb96/EEG_ClientGUI/commits/master](https://github.com/dorianb96/EEG_ClientGUI/commits/master)
- [https://github.com/dorianb96/Cloudera_Quickstart_Install/commits/master](https://github.com/dorianb96/Cloudera_Quickstart_Install/commits/master)
- [https://github.com/dorianb96/kerberos-on-cloudera-quickstart/commits/master](https://github.com/dorianb96/kerberos-on-cloudera-quickstart/commits/master)
- [https://github.com/dorianb96/gsoc-architecture-explained/commits/master](https://github.com/dorianb96/gsoc-architecture-explained/commits/master)
