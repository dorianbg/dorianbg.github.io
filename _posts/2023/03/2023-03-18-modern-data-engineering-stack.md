---
title: "Modern data engineering stack"
date: "2023-03-18"
categories: 
  - "data-engineering"
  - "data-systems"
  - "data-warehousing"
---

### **Data storage**

- **Data lake: Object store + storing data in a columnar format**
    - examples:
        - S3 + Athena (Presto)
    - PROs:
        - cheaper than a data warehouse
        - easier to store very complicated data (eg. arrays in a column) than in most data warehouses
    - CONs
        - worser performance than a data warehouse
        - can become very messy without a good data catalog and understanding of schemas
    - I would always choose a data warehousing solution over Parquet files in Object storage, though Apache Iceberg is a very promising solution in this arena...

- **Data warehouse**
    - example:
        - **Snowflake**
        - Google BigQuery
        - Amazon Redshift
        - Click House (great solution but not ANSI SQL compliant and with disappointing window functions support in 2023!)
    - PROs
        - very fast and efficient
    - CONs
        - more expensive storage
        - less flexible schemas
    - Note
        - prefer data warehouses where the storage is fundamentally decoupled from the compute (eg. use Snowflake and not Redshift)

- Things to avoid:
    - using a traditional RDBMS (SQL Server, Oracle, PostgreSQL) for storing data on which only analytics will be done
        - the traditional databases do not scale well (though if you have smaller data I would still entertain them) and are just not optimised for this use case no matter how many optimisation they now have (parallel execution, materialized views, column store indexes, roll-ups using upserts, B+ tree indexes...)

### **Data ingestion**

- ELT vs ETL
    - Always prefer the ELT approach as data warehouses
        - with an ELT approach you are more flexible, the loading process is faster and data warehouses are better at doing transformations (in a batch manner) compared to ETL tools
- Depending on the data source
    - Is the data source from outside of your organisation and somewhat "popular" (e.g. stripe, adyen, blaze...)?
        - Easy solution: use companies that have build these connectors e.g. Fivetran, Stitch... though the final decision depends on your budget
    - On-premise, custom data source
        - write your own connectors, probably simplest in Python

**Tasks scheduling**

- Ideally you could use an automated data movement platform (e.g. Fivetran stitch, meltano...) for most of the data ingestion tasks
- For other custom tasks I like to use **Airflow**, ideally in a cloud setting (so that Airflow is managed by the cloud provider) and that is uses **Kubernetes executors** (which allow it to very easily scale)
- For managing and executing SQL views and transformations use **DBT**. It's a phenomenal tool.
