---
title: "My view on responsibilities of a modern data engineer"
date: "2020-01-25"
categories: ["data-engineering"]
tags: ["principles"]
---

In my opinion one of the main responsibilities of a data engineer is to build the infrastructure and clean data for efficient use by various stakeholders (analysts, data scientists...).

Data platform engineering are also focused on building tooling that can be used by other engineers or data scientists.

#### Core responsibilities of an analytics focused data engineer are:

- **Setting up and managing the data warehouse (including ensuring HA, backups...)**
    - requires a good knowledge of databases and their internals
    - most of the textbooks are outdated as they mainly cover traditional RDMBS systems but there is a [great course](https://15721.courses.cs.cmu.edu/spring2020/) by Andy Pavlo from CMU
        
- **Setting up and managing up the data infrastructure (messaging systems, object storage, compute engines...)**
    - requires knowledge of cloud vendors tools internals (eg. how is Kinesis different than Kafka?), infrastructure provisioning (Terraform, Ansible) and potentially tools like Kubernetes for custom applications
- **Designing a high-level architecture of the data warehouse and writing guidelines for managing the contents**
    - requires a lot of prior experience and business specific knowledge
    - a good source is the book [The Data Warehouse Toolkit](https://www.amazon.co.uk/Data-Warehouse-Toolkit-Complete-Dimensional/dp/0471200247) by Ralph Kimball though some concepts might be outdated (handling of slowly changing dimensions)
        
- **Handling performance issues in the data access patterns of end users** 
    - requires a good knowledge of data systems (ie. messaging queues, caches, databases...)
    - a good source is [Designing Data-Intensive Applications](https://www.goodreads.com/en/book/show/23463279) by Martin Kleppmann
        
- **Ingesting data into the data warehouse**
    - whilst a data engineer should be very good at this, the real goal is to empower the end-users to do this themselves by using (or writing) frameworks that allow declarative definitions of data pipelines
    - a data engineer should know when to use tools like [Fivetran](https://fivetran.com/), [Alooma](https://www.alooma.com/)... (which should be preferred for modern SaaS data sources) to ingest data and when roll up one's sleeves
    - Maxime Beauchemins' [blogs posts](https://medium.com/@maximebeauchemin/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a) on functional data engineering are a good source of information on this
- **Transforming the data in the data warehouse**
    - whilst a data engineer should be very good at this, the real goal is to empower the end-users to do this themselves using tools like [dbt](https://github.com/fishtown-analytics/dbt)
    - a great book on SQL is [T-SQL Querying by Itzik Ben-Gan](https://www.goodreads.com/book/show/25023957-t-sql-querying)
        
- **Setting up and managing production workloads and the related infrastructure**
    - examples:
        - setting up Apache Superset along with all the security, permission and other concerns
        - setting up an internal python package repository
        - creating build pipelines (eg. in Jenkins)
        - managing the production job scheduler (eg. Airflow)

#### DE tasks often overlap with the usual back-end (and dev-ops) tasks:

- **Back-end development**
    - building back-end APIs to serve data (eg. using Flask or Spring Boot)
        - usually this won't be for a typical "operational" front-end system
    - managing streaming data (eg. consuming Kafka messages in real-time and acting on them)
    - building a caching layer (Redis, Plasma cache...)
    - building a search layer (Elastic search, Apache solr...)
- **Dev-ops**
    - managing infrastructure, builds..

#### A data engineer should also be able to do basic analytics:

- **Analytics and Business intelligence**
    - writing queries to answer complicated business questions
    - composing accurate reports
- **Data science**
    - non-critical predictive workloads (ie. running a simple classifier or a regression task)
- **Front-end development (eg. R shiny, dash, D3.js, React.js...) or creating dash boards using a tool (Tableau...)**
