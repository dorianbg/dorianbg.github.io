---
title: "Data extraction and transformation design patterns"
date: "2023-03-18"
categories: ["data-engineering"]
tags: ["principles"]
---

- You will need to choose between full and incremental extracts
    - Always prefer full extracts and partitioned snapshots if the data is small enough
    - If this is not possible, then design an incremental approach

- Extract data in an idempotent manner
    - Eg. extract data for a specific timeframe from the data source and make sure that in your data store you have the same data in the same timeframe (eg. same size, distinct values in columns…)
    - Eg. perform incremental extracts using row IDs,so if you have a set of rows in the data source then do a set difference to determine which rows should be ingested
        - Otherwise maybe incrementally ingest using the created/modified timestamps but only if they are reliable. After this extraction window functions are necessary for data deduplication (choosing the latest source)

- Use a staging table for the output of ETL job
    - Eg. if you recompute an output table, first create the temporary output table, then check it’s values are good before replacing the actual output table

- Use partitions for per-partition data extraction
    - Eg. an ETL gets data every 1 hour or 1 day from the data source and puts it into the desired output

- Build a dashboard to visualise input data 

- Reproducibility through immutable datasets
    - Snapshot your extracts and dimensions
    - Have a clear data lineage

- Favor ELT over ETL
    - Put work into the database engine
    - Have a specific ingestion staging area to put data into

- Favor SQL over anything else for transformation

- Add data quality tests and measures during every ingest
    - If the data quaity of some source system is below a threshold, throw that data away

- Create data ingestion SLAs
    - A script that checks the latest timestamp for some data
