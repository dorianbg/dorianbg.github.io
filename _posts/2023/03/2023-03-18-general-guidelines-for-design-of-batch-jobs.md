---
title: "General guidelines for design of batch jobs"
date: "2023-03-18"
categories: ["data-engineering"]
tags: ["principles"]
---

- **Have a config driven and easily extensible library for your ETL**
    - You can easily add new extractors and your existing loaders will take care of loading the data.
    - Important to abstract away DB specific operations and just work with higher level abstractions (e.g. dataframes)
    - Great example: [https://github.com/amundsen-io/amundsendatabuilder](https://github.com/amundsen-io/amundsendatabuilder)
    - Something like this as well [https://github.com/YotpoLtd/metorikku](https://github.com/YotpoLtd/metorikku)

- **Break the process down into components**
    - UNIX way of doing
    - Eg first step is ingestion, second is taking the ingested data and filtering...

- **Idempotent and deterministic ETL tasks**
    - Have no side effects
    - Use immutable storage
    - Usually target a single partition
    - Never UDPATE, INSERT, DELETE  (mutations)
    - Limit the number of source partitions (no wide-scans)
    - Transform your data, don’t overwrite
    - All jobs must be idempotent meaning that no matter how many times you run it, the state after N runs is same as after N + M runs

- **Use configuration files to define your ETL**
    - Do not code to a specific data source, query, file format, location…
    - Code the dependencies of the job as external file
        - eg. inputs, outputs, output file format...

- **Engineer to anticipate failures**
    - Use retries (potentially exponential)
        - Expect external connections to fail 
        - Expect data not to be there

- **Workflow tool you use needs to have:**
    - Basic dependency management
    - Clarity on status
    - Scheduling (eg. similar to crontab)
    - Easy access to logs
    - Parametrized retries
    - Notifications (errors, success)
    - Retries of jobs

- **Make sure your code base handles:**
    - Testing - test the code, especially have integration tests
    - Logging - log everything (ideally ship the code)
    - Packaging
